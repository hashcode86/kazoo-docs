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

## Message Format
Một ví dụ khác về cuộc gọi:
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

## Transfer reason & các biến cần set cho ghi nhận Route-Source

module | reason |  set_variables trước khi transfer | Description
--- | --- | --- | ---
transfer | auto_transfer | auto_transfer_from |
transfer | bot_transfer | referred_callbot, referred_callbot_by, referred_callbot_time |
transfer | queue_timeout |  |

## Các loại route source

