# Route Tracking
## Khái niệm callflow route
Để mô tả về `cf_route_id`, ta lấy ví dụ cụ thể về một cuộc gọi được xử lý bởi hệ thống như sau: 
* (1) Cuộc gọi tới đầu số `voice_channel_number` của một dịch vụ
* (2) Cuộc gọi auto_transfer (blind transfer) qua một đầu số `voice_channel_number` khác
  * Cuộc gọi được vào queue được agent phục vụ
* (3) Agent thực hiện blind transfer cuộc gọi tới một đầu số IVR khác
  * khách hàng bấm dtmf theo kịch bản của ivr để vào queue
  * cuộc gọi tới pivot của một `voice_channel_number` để vào queue tiếp
  
Đoạn (1), (2), (3) ở trên được gọi là 3 callflow route (có cf_route_id khác nhau cho từng đoạn). Mục
đích của việc quản lý thành nhiều callflow route như này để có thể tracking được cuộc gọi theo từng phân đoạn, ví dụ:

* Tại (2) - cf_route_id=2
  * Cuộc gọi tới (2) do `auto_transfer` và đến từ `voice_channel_number` của (1)
  * Cuộc gọi khi vào queue ở (2), thì CALL_OVERFLOW cũng có thông tin về `auto_transfer`
* Tại (3) - cf_route_id=3
  * Cuộc gọi tới (3) do `agent_transfer` và đến từ `voice_channel_number` của (2)
  * Tại IVR khách hàng vào MENU & bấm DTMF => có gắn cùng với cf_route_id=3
  * Cuộc gọi vào queue ở (3), thì CALL_OVERFLOW có thông tin về `agent_transfer` đến & last menu, dtmf của khách hàng trước khi vào queue là gì.

## Callflow Route Message
### callflow_route
```json
{
    "Old-Callflow-Route-ID": "1727799112345-b41fee8c",
    "Callflow-Route-Source": {
        "type": "auto_transfer",
        "auto_transfer_from": "30000"
    },
    "Unix-Time-Ms": 1727799112449,
    "Request": "10000@10.10.10.1",
    "Callflow-Route-ID": "1727799112449-c746d70c",
    "Call-ID": "zYV~y2vH5k",
    "Account-ID": "e7339ac33c3b5027922fde17b88eb772",
    "Event-Name": "callflow_route",
    "Event-Category": "notification",
}
```

| Field                 | Type   | Nullable | Description                                                      |
| --------------------- | ------ | -------- | ---------------------------------------------------------------- |
| Account-ID            | String | No       |                                                                  |
| Call-ID               | String | No       | Id cuộc gọi, một cuộc gọi có thể có nhiều bản tin callflow route |
| Event-Name            | String | No       | callflow_route                                                   |
| Callflow-Route-ID     | String | No       | id route hiện tại                                                |
| Unix-Time-Ms          | Number | No       | thời gian bắt đầu của route                                      |
| Old-Callflow-Route-ID | String | Yes      | id của route trước khi vào route hiện tại                        |
| Callflow-Route-Source | Object | Yes      | thông tin cuộc gọi vào route hiện tại như nào                    |
### Callflow-Route-Source
Các loại của Callflow Route Source:
* ivr: ví dụ, cuộc gọi đến đầu số ACD từ IVR
* auto_transfer
* bot_transfer
* queue_timeout
* agent_transfer
* smart_routing
* ...
#### Callflow-Route-Source: IVR

| Field                   | Type   | Description                   |
| ----------------------- | ------ | ----------------------------- |
| type=ivr                | String |                               |
| ivr_call_id             | String | Id của cuộc gọi trên zone IVR |
| ivr_call_request_number | String | đầu số trên zone IVR          |
| ivr_last_menu_id        | String | menu cuối cùng trên zone ivr  |
| ivr_last_menu_digits    | Number | Phím cuối cùng trên zone ivr  |

Setup: Cuộc gọi từ zone IVR qua ACD cần có CSH như dưới
```json
{
    "children": {},
    "data": {
        "use_local_resources": true,
        "to_did": "10000",
        "custom_sip_headers": {
            "P-IVR-Call-ID": "{sip_call_id}",
            "P-IVR-Call-Request": "{sip_req_user}"
        }
    },
    "module": "resources"
}
```

#### Callflow-Route-Source: Smart Routing

| Field              | Type   | Description                   |
| ------------------ | ------ | ----------------------------- |
| type=smart_routing | String |                               |
| smart_routing_from | String | Id của cuộc gọi trên zone IVR |
| smart_routing_to   | String | đầu số trên zone IVR          |
|                    |        |                               |
Setup:
* Cuộc gọi từ zone acd gốc, cần cho qua flow như dưới để set 2 CSHs và tới module resources 
```json
{
    "data":{
        "to_did":"10000",
        "use_local_resources":true,
        "dynamic_flags":[
            "zone"
        ],
        "custom_sip_headers":{
            "P-Smart-Routing-To":"TARGET_QUEUE_ID_HERE",
            "P-Smart-Routing-From":"SRC_QUEUE_ID_HERE"
        }
    },
    "module":"resources"
}
```

* API tràn tới flow. với smart_routing=true và tại queue gốc có abandoned_reason=smart_routing_exit
```shell
curl -v -X POST \
    -H "Content-Type: application/json" \
    -H "X-Auth-Token: {AUTH_TOKEN}" \
    -d '{"data":{"action":"overflow","smart_routing": true, "flow":JSON_FLOW_NHU_TREN}}' \
    http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/channels/{UUID}
```
#### Callflow-Route-Source: auto_transfer

| Field                | Type   | Description                                     |
| -------------------- | ------ | ----------------------------------------------- |
| type=auto_transfer   | String |                                                 |
| auto_transfer_from   | String | Id của đối tượng thực hiện auto_transfer        |
| voice_channel_number | String | vc number của đối tượng thực hiện auto_transfer |
Setup: callflow thực hiện auto_transfer tới target 10000, có thể set custom_application_vars ngay tại module `transfer` (nên có thể không cần cho qua `set_variables` trước nó) 
```json
{
    "data": {
        "target": "10000",
        "transfer_type": "blind",
        "reason": "auto_transfer",
        "custom_application_vars": {
            "auto_transfer_from": "30000"
        }
    },
    "module": "transfer"
}
```

#### Callflow-Route-Source: bot_transfer

| Field                | Type   | Description                              |
| -------------------- | ------ | ---------------------------------------- |
| type=bot_transfer    | String |                                          |
| callbot_name         | String | tên của bot được chuyển tới              |
| callbot_by           | String | source queue nào chuyển cuộc gọi tới bot |
| callbot_time         | number | thời điểm cuộc gọi được chuyển tới bot   |
| voice_channel_number | String | vc number của dịch vụ trước khi vào bot  |

Set biến cho cuộc gọi trước khi kết nối tới bot
```json
{
    "data": {
        "custom_application_vars": {
            "referred_callbot": "bot_name_here",
            "referred_callbot_time": "1720523083000",
            "referred_callbot_by": "ID_SRC_QUEUE_HERE"
        }
    },
    "module": "set_variables",
    "children": {
        "_": {FLOW_TO_CALLBOT_HERE}
    }
}
```

API được call bởi bot (hoặc ccms gọi vào kazoo) để chuyển cuộc gọi lại một đầu số ACD
```shell
curl --location 'http://{API_SERVER}/v2/accounts/{ACCOUNT_ID}/channels/{CHANNEL_ID}' \

--header 'Content-Type: application/json' \
--header 'X-Auth-Token: ${AUTH_TOKEN}' \
--data '{
    "data": {
        "action": "blind_transfer",
        "target": "20000",
        "reason": "bot_transfer",
        "custom_application_vars": {
            "cac_bien_do_bot_set" : ""
        }
    }
}'
```

Callflow route khi cuộc gọi tới đầu số 20000 sẽ được ghi nhận là bot_transfer tới

#### Callflow-Route-Source: queue_timeout
Cuộc gọi timeout ở queue và chuyển tới đầu số đích, ví dụ 10000
```json
{
    "data": {
        "target": "10000",
        "transfer_type": "blind",
        "reason": "queue_timeout",
        "custom_application_vars": {
        }
    },
    "module": "transfer"
}
```
#### Callflow-Route-Source: agent_transfer
agent tranfer cuộc gọi vào đầu số IVR hoặc ACD khác, thực hiện gọi Channel api với:
* target: đầu số ivr hoặc acd đích
* reason: agent_transfer
(tương tự bot_transfer)

## Ví dụ cuộc gọi có một route & trong route này có tới queue

* Cuộc gọi tới đầu số 30000 và được `auto_transfer` tới đầu số 10000
* Tại đầu số 10000, cuộc gọi vào menu, khách hàng bấm 1 để vào Queue
### Setup
* Callflow của đầu số 30000
  * Biến `auto_transfer_from` như thông thường
  * Data của module `transfer` có thêm `"reason": "auto_transfer"`
```json
{
    "data": {
        "custom_application_vars": {
            "auto_transfer_from": "30000"
        }
    },
    "module": "set_variables",
    "children": {
        "_": {
            "data": {
                "transfer_type": "blind",
                "target": "10000",
                "reason": "auto_transfer"
                // hoặc có thể đặt custom_application_vars ở đây
            },
            "module": "transfer"
        }
    }
}
```
* Callflow của đầu số 10000
```json
{
    "data": {
        "custom_application_vars": {
            "keep_source_moh_list": "fae73605016300b333563d8a39dea6bb"
        }
    },
    "module": "set_variables",
    "children": {
        "_": {
            "data": {
                "id": "1084e4f782ca5f2e34beee6b7bfa0b9c"
            },
            "module": "menu",
            "children": {
                "1": {
                    "data": {
                        "id": "32f0dfe78df5c7945febdff0b9dd64e9"
                    },
                    "module": "acdc_member",
                    "children": {}
                }
            }
        }
    }
}
```
### Messages
* menu_digits
```json
{
    "Callflow-Route-ID": "1727615096508-87f3eff8",
    "Remaining-Retry-Count": 2,
    "Digits-Matched": true,
    "Digits": "1",
    "Unix-Time-Ms": 1727615102960,
    "Menu-Unix-Time-Ms": 1727615096566,
    "Menu-ID": "1084e4f782ca5f2e34beee6b7bfa0b9c",
    "Request": "10000@10.10.10.1",
    "Call-ID": "rHxskTXWU6",
    "Account-ID": "e7339ac33c3b5027922fde17b88eb772",
    "Event-Name": "menu_digits"
}
```
    Callflow-Route-ID: id của route hiện tại
    Request: route hiện tại đang request tới 10000 (trước đó là 30000)

* acdc_call_stat.waiting
```json
{
    "Custom-SIP-Headers": {},
    "Custom-Application-Vars": {},
    "Overflow-ID": "1727615102981-eb56c2f5",
    "Overflowing": false,
    "Overflow": {
        "Overflow-ID": "1727615102981-eb56c2f5",
        "Overflowing": false,
        "Queue-Unix-Time-Ms": 1727615102981,
        "Voice-Channel-Number": "33366",
        "Request": "10000@10.10.10.1",
        "Last-Menu-ID": "1084e4f782ca5f2e34beee6b7bfa0b9c",
        "Last-Menu-Digits": "1",
        "Callflow-Route-ID": "1727615096508-87f3eff8",
        "Callflow-Route-Source": {
            "type": "auto_transfer",
            "auto_transfer_from": "30000"
        }
    },
    "Request": "10000@10.10.10.1",
    "Unix-Time-Ms": 1727615102986,
    "Queue-ID": "b0794e7ca7baf15832935ca65a08ee6e",
    "Account-ID": "e7339ac33c3b5027922fde17b88eb772",
    "Call-ID": "rHxskTXWU6",
    "Event-Name": "waiting",
    "Event-Category": "acdc_call_stat"
}
```
    Overflow.Callflow-Route-ID: Bằng với Callflow-Route-ID ở menu
    Overflow.Last-Menu-*: Menu và phím bấm gần nhất trước khi vào queue
    Overflow.Callflow-Route-Source: Nguồn của cuộc gọi trước khi tới Route hiện tại


