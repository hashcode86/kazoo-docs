# Callflow
## set_variables
Cấu hình | Type | Default | Description
--- | --- | --- | ---
custom_application_vars | Object | {} | Danh sách các biến cần được set thêm vào Call
overwrite_existing | Boolean | true | Nếu Call đã có biến được khai báo ở `custom_application_vars` thì có set đè giá trị vào hay không

```json
{
    "module": "set_variables",
    "data": {
        "overwrite_existing": false,
        "custom_application_vars": {
            "var_name": "new_value"
            //overwrite_existing=false và nếu Call đã có biến var_name thì khai báo ở đây sẽ được bỏ qua
        }
    }
}
```

## acdc_member

### Set Moh & Distribution delay theo từng khách hàng
```json
{
    "data": {
        "id": "your_queue_id",
        "distribution_delay": 10, //seconds
        "moh_list": ["2796b13d2b92b5f723ddf23905bbd2c6", "34132c66ea1e771c63bf9e0d544bbb4e"]
    },
    "module": "acdc_member",
    "children": {}
}
```

# Queue Features
## Queue setting

Cấu hình | Type | Default | Description
--- | --- | --- | ---
voice_channel_number | String | null | đầu số của nhóm queue chứa queue hiện tại
moh_list | List | null | danh sách nhạc chờ trong queue
moh_strategy | String | null | "sequential", "random"
distribution_delay | Integer | 0 | Đặt giá trị, ví dụ 10 nếu yêu cầu khách hàng nghe nhạc chờ (truyền thông) tối thiểu 10s trước khi phân phối tới agent
hold_medias | List | [] | Nếu hold_medias là null hoặc [] thì sẽ sử dụng âm hold máy theo setting của account/user/device
hold_medias_strategy | String | "random" | Hoặc "sequential" nếu muốn âm hold máy được play theo thứ tự
last_called_agent_routing | Boolean | false | Tắt / bật  phân phối tới last agent
record_extension | String | "mp3" | "mp4" - ghi hình
record_stereo | Boolean | false | Tắt / bật  ghi âm đa kênh
delay_200_ok | Boolean | false | Đặt bằng true nếu cần cuộc gọi vào queue, gặp agent mới tính cước viễn thông của khách hàng. Queue callback cũng cần đặt `true` để khi agent nhấc máy mới call tới khách hàng qua click2call api
secondary_agents | List | [] | Cấu hình danh sách các agent có skill thấp hơn, và cũng là ưu tiên thấp hơn khi nhận cuộc gọi (2 mức skill). Agent phải được gán vào queue thì cấu hình agent đó ở danh sách này mới có ý nghĩa.

## Tràn Queue
### Mô tả
* Cuộc gọi từ ivr, tới queue gốc `Queue0`, vào queue tại thời điểm `Time0`, được phát nhạc chờ `Media0` của  Queue0
* Tại thời điểm `Time1`, hệ thống quyết định tràn cuộc gọi tới `Queue1`, các tham số được áp cho cuộc gọi khi vào `Queue1`:
  * Thời điểm vào `Queue1` được set là `Time0`, do đó thời gian cuộc gọi còn lại ở `Queue1` là: `Queue1.connection_timeout - (Time1 - Time0)`
  * Nhạc chờ khi tại `Queue1` được giữ là `Media0`
* Tại thời điểm `Time2`, nếu tiếp tục tràn cuộc gọi tới  `Queue2`, các tham số của `Queue0` vẫn tiếp tục được áp dụng

### Api
Api dưới đây cho phép tràn cuộc gọi của khách hàng tới một callflow được định nghĩa trước
```shell
curl -v -X POST \
    -H "Content-Type: application/json" \
    -H "X-Auth-Token: {AUTH_TOKEN}" \
    -d '{"data":{"action":"overflow","callflow_id":"target_callflow_id_here"}}' \
    http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/channels/{UUID}
```

Key | Description | Type | Default
--- | --- | --- | ---
UUID | id của call channel cần thực hiện tràn (channel khách hàng) | String |
callflow_id | id của callflow mà khách hàng sẽ được tràn tới | String |
flow | flow mà khách hàng sẽ được tràn tới | json | null

## Smart Routing
Tương tự như tràn, tuy nhiên smart routing sẽ là đưa cuộc gọi tới module `resource` để cuộc gọi được chuyển qua miền khác
* Đầu tiên, cần xác định chuyển tới queue đích nào, đầu số nhóm queue đích để có flow json như dưới:
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

Key | Description | Type | Default
--- | --- | --- | ---
UUID | id của call channel cần thực hiện tràn (channel khách hàng) | String |
callflow_id | id của callflow mà khách hàng sẽ được tràn tới | String |

## Nhạc chờ custom
* Nhạc chờ trong queue được setting bằng thuộc tính `moh_list`. Nhạc chờ set tại đây sẽ là danh sách file mặc định và không có khả năng tùy biến theo user
* Một số tình huống cần set đè lại cấu hình moh_list ở trên (phát nhạc chờ theo tỉnh, nhạc chờ riêng theo nhóm khách hàng), có thể thực hiện theo 2 cách:
```json
{
    "data": {
        "id": "your_queue_id",
        "moh_list": ["2796b13d2b92b5f723ddf23905bbd2c6", "34132c66ea1e771c63bf9e0d544bbb4e", "5d1713a3d8c52e44c7cef1168d3c4801"]
    },
    "module": "acdc_member",
    "children": {}
}
```
* Khi auto_transfer cuộc gọi tới một queue khác (queue đích) và muốn tại queue đích vẫn play nhạc chờ của queue nguồn:
```json
{
    "data": {
        "flow": {
            "module": "set_variables",
            "data": {
                "overwrite_existing": false, // nếu cuộc gọi đã có các biến bên dưới thì biến dưới sẽ không được set vào cuộc gọi
                "custom_application_vars": {
                    "keep_source_moh_list": "fae73605016300b333563d8a39dea6bb" // id queue nguồn
                }
            },
            "children": {
                "_": {
                    "data": {
                        "id": "32f0dfe78df5c7945febdff0b9dd64e9" // id callflow queue đích
                    },
                    "module": "callflow",
                    "children": {}
                }
            }
        }
    }
}
```
Note: Khi cuộc gọi đã tới được queue (`cf_acdc_member`), thì variable `keep_source_moh_list` sẽ được remove khỏi cuộc gọi. Việc này là cần thiết vì nếu cuộc gọi có được transfer tới một queue nào đó khác thì ko bị vô tình play lại nhạc chờ theo biến `keep_source_moh_list`

## Call Transfer Tracking
### Kết thúc do transfer
* Cuộc của member (member_call) được agent trả lời, khi agent transfer cuộc gọi đi, bổ sung trường `Referred-To` cho record acdc_stats.call_processed thể hiện cuộc gọi được transfer tới đích nào
* acdc_stats.call_processed có `Referred-To=<sip:20000@test.mycc.vn:5060` và `Hung-Up-By=agent` và như thế được hiểu cuộc gọi được kết thúc bởi agent và agent transfer member_call tới một đích mới
```json
{
    "Referred-To":"<sip:20000@test.mycc.vn:5060>",
    "Hung-Up-By":"agent",
    "Processed-Timestamp":63887590276,
    "Agent-ID":"8b14e6242b49c65f68b85a7f4075a989",
    "Unix-Time-Ms":1720371076361,
    "Timestamp-Milliseconds":63887590276361,
    "Queue-ID":"b0794e7ca7baf15832935ca65a08ee6e",
    "Account-ID":"e7339ac33c3b5027922fde17b88eb772",
    "Call-ID":"1pLhRrTghw",
    "Node":"kazoo_apps@z0kapps-0.z0nodes.default.svc.cluster.local",
    "Msg-ID":"f4bb070702d12905",
    "Event-Name":"processed",
    "Event-Category":"acdc_call_stat",
    "App-Version":"4.0.0",
    "App-Name":"acdc"
}
```

### Được chuyển đến do transfer
* Yêu cầu:
  * Đầu số của nhóm queue (kênh) cần set cho tất cả các queue trong nhóm, field: `voice_channel_number`
  * Cuộc gọi được chuyển đến nhóm queue qua 2 cách là module transfer hoặc api blind_transfer, với target là đầu số của nhóm queue


* Cuộc gọi của khác hàng khi tới một queue thì có thể đến từ `ivr`, `auto_transfer`, `agent_transfer` hoặc `bot_transfer`
* **agent_transfer** là khi cuộc gọi đã được một agent trả lời ở src_queue, sau đó agent thực hiện transfer tới đầu số nhóm queue hiện tại
* **bot_transfer** là khi cuộc gọi của khách hàng trước đã vào bot (do src_queue cấu hình vào bot), và bot transfer tới đầu số nhóm queue hiện tại
* **auto_transfer** là khi cuộc gọi của khách hàng muốn vào src_queue, nhưng src_queue cấu hình auto_transfer tới đầu số nhóm queue hiện tại

#### agent_transfer
Agent trả lời cuộc gọi, sau đó transfer tới một kênh khác (ví dụ 20000) thì khi cuộc gọi tới kênh mới và vào một queue của kênh này thì bản tin acdc_stats.call_waiting sẽ có:
* **Source-Type**: agent_transfer
* **Source-ID**: id của queue ban đầu

#### auto_transfer
Cuộc gọi khi auto_transfer tới một nhóm queue đích (ví dụ 20000) thì cần được settting như sau:
```json
{
    "data": {
        "custom_application_vars": {
            "auto_transfer_from": "ID_SRC_QUEUE_HERE"
        }
    },
    "module": "set_variables",
    "children": {
        "_": {
            "data": {
                "transfer_type": "blind",
                "target": "20000"
            },
            "module": "transfer"
        }
    }
}
```
* `auto_transfer_from`: đặt là giá trị id queue gốc, thể hiện cuộc gọi đang được auto_transfer từ queue nào
* `target`: Đầu số nhóm queue sẽ chuyển tới

Khi cuộc gọi tới kênh 20000 và vào một queue của kênh này thì bản tin acdc_stats.call_waiting sẽ có:
* **Source-Type**: auto_transfer
* **Source-ID**: id của queue gốc thực hiện auto_transfer đi

### Chuyển cuộc gọi đến Callbot & nhận lại từ callbot
Khi cuộc gọi vào một queue nhưng queue này có cấu hình chuyển bot thì flow cho việc chuyển bot:
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

Bot chuyển cuộc gọi lại tới một nhóm queue (ví dụ 20000, qua api hoặc sip refer), và tại nhóm 20000, cuộc gọi vào một queue của kênh này thì bản tin acdc_stats.call_waiting sẽ có:
* **Source-Type**: bot_transfer
* **Callbot**: tên của callbot trước đó
* **Callbot-By**: id của queue trước khi vào bot
* **Callbot-Time**: thời gian vào bot trước đó

### Hot transfer
Để ghi nhận CDR cuộc gọi vào queue (CALL_QUEUE) với hiện trạng là `hot transfer`, trước khi vào queue đích thì set thêm biến `member_flags` để đánh dấu
```json
{
    "data": {
        "custom_application_vars": {
            "member_flags": "hot_transfer"
        }
    },
    "module": "set_variables",
    "children": {
        "_": {FLOW_TO_QUEUE_HERE}
    }
}
```

* Khi vào queue (cf_acdc_member) thì bản tin acdc_call_stat.waiting sẽ có trường `member_flags`, cần check giá trị của nó liệu có chưa giá trị `hot_transfer` không để ghi nhận
* Sau khi qua cf_acdc_member thì CCVs của Call sẽ không còn biến `member_flags` nữa, nên sau đó nó có vào queue mới thì cũng không có cờ được đẩy ra (trừ khi được set tiếp)
* CAVs của call thì vẫn sẽ còn `member_flags`

# CHANNEL API

## meta listen_on=peer

##Blind Transfer
Use case: Khách hàng đang nói chuyện với callbot & callbot cần thực hiện transfer cuộc gọi của khách hàng về một đầu số dịch vụ:

```shell
curl -v -X POST \
    -H "Content-Type: application/json" \
    -H "X-Auth-Token: {AUTH_TOKEN}" \
    -d ' {
   "data": {
     "action": "blind_transfer",
     "target": "1001"
   }
 }' http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/channels/{UUID}
```

Key | Description | Type | Default
--- | --- | --- | ---
UUID | id của customer call channel cần thực hiện transfer | String |
target | Extension/DID của dịch vụ | String |

## Intercept
### Use case
Một cuộc gọi khi đang đàm thoại sẽ, có 2 legs: A-leg (khách hàng) và B-leg (agent A1).

`intercept` là api cho phép tạo một leg mới (C-leg), ring tới một agent (A2) được chỉ định. Khi A2 nhấc máy (C-leg ở trạng thái Answer) thì B-leg kết thúc, A-leg được bridge với C-leg để A2 đàm thoại với khách hàng.

Dùng channel `intercept` API để implement 2 nghiệp vụ chính sau:
* Agent (A1) transfer cuộc gọi của khách hàng cho một agent khác (A2): A1 gọi API này để ring cuộc gọi tới A2; A1 và khách hàng vẫn đàm thoại bình thường cho tới khi A2 nhấc máy.
* Giám sát viên (S1) cướp cuộc gọi đang đàm thoại của A1 và khách hàng: S1 gọi API này để ring cuộc gọi tới mình; S1 nhấc máy thì A1 và khách hàng bị ngắt luồng đàm thoại

### API
```shell
curl -v -X POST -H "Content-Type: application/json" -H "X-Auth-Token: {AUTH_TOKEN}" -d '
{
    "data": {
        "action": "intercept",
        "target_type":"user",
        "target_id": "548806d6bae48e98d3da4167e2c9868e",
        "unbridged_only": false,
        "intercept_type": "transfer"
    }
}
' http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/channels/{UUID}
```

Key | Description | Type | Default
--- | --- | --- | ---
UUID | ID của customer channel cần thực hiện intercept | String |
target_type | "user" or "device" | String |
target_id | user id hoặc device id được nhận cuộc gọi | String |
intercept_type | "transfer" or "intercept", dùng đánh dấu cuộc gọi chuyển đi do cướp hay transfer thông thường| String

Cuộc gọi của khách hàng đang trong queue, khi bị intercept thì bản tin `processed` có thêm 3 trường `Intercept-*` như dưới:
```json
{
"Intercepted-Type":"intercept",
"Intercepted-To-User":"548806d6bae48e98d3da4167e2c9868e",
"Intercepted-To":"1001",
"Hung-Up-By":"agent",
"Processed-Timestamp":63888698282,
"Agent-ID":"8b14e6242b49c65f68b85a7f4075a989",
...
}
```

## Store Recording
Lấy file ghi âm (khi cuộc gọi đang diễn ra) và lưu vào một http endpoint (url) qua method HTTP PUT

* Request
```shell
curl -v -X POST -H "Content-Type: application/json" -H "X-Auth-Token: {AUTH_TOKEN}" -d '
{
    "data": {
        "action": "store_recording",
        "url":"your-url-here"
    }
}
' http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/channels/{UUID}
```
* Response
```json
{
"media_name": "548806d6bae48e98d3da4167e2c9868e.mp3",
"url":"your-url-here"
}
```

Key | Description | Type | Default
--- | --- | --- | ---
UUID | ID của customer channel cần thực hiện lấy file ghi âm | String |
url | Địa chỉ của HTTP Endpoint sẽ nhận file ghi âm qua http put | String |
media_name | Tên của file ghi âm cho cuộc gọi hiện tại | String |

<del>
## Attended Transfer
Khách hàng đang nói chuyện với agent1 và cuộc gọi cần chuyển tới agent2, api này dùng để:
* agent1 chọn chuyển tới agent2
* customer được nghe MOH, một cuộc gọi được ring tới agent2
* agent2 không nhấc máy, customer kết nối lại agent1
* agent2 nhấc máy, agent1 và agent2 trao đổi thông tin trước
  * agent2 ngắt máy (không đồng ý nhận), customer kết nối lại agent1
  * agent1 ngắt máy (agent2 đồng ý nhận), customer kết nối với agent2

```shell
curl -v -X POST \
    -H "Content-Type: application/json" \
    -H "X-Auth-Token: {AUTH_TOKEN}" \
    -d ' {
   "data": {
     "action": "transfer",
     "target": "1001",
     "transfer-type": "attended",
     "leg": "self"
   }
 }' http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/channels/{UUID}
```

Key | Description | Type | Default
--- | --- | --- | ---
UUID | id của customer call channel cần thực hiện transfer | String |
target | Extension/DID của agent2 | String |
</del>

# Outbound Status
## Login ở chế độ outbound
```shell
curl -v -X POST \
    -H "Content-Type: application/json" \
    -H "X-Auth-Token: {AUTH_TOKEN}" \
    -d ' {
    "data":{
        "status":"login",
        "outbound_login": true
    }
 }' http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/agents/{AGENT_ID}/status
```

## Các bản tin trạng thái chế độ outbound
  * Agent vừa login với chế độ outbound
```json
[
    {
        "Unix-Time-Ms":1722652319923,
        "Agent-ID":"8b14e6242b49c65f68b85a7f4075a989",
        "Account-ID":"e7339ac33c3b5027922fde17b88eb772",
        "Event-Name":"logged_in",
        "Event-Category":"acdc_status_stat",
    },
    {
        "Pause-Alias":"outbound",
        "Pause-Time":"infinity",
        "Unix-Time-Ms":1722652324823,
        "Agent-ID":"8b14e6242b49c65f68b85a7f4075a989",
        "Account-ID":"e7339ac33c3b5027922fde17b88eb772",
        "Event-Name":"paused",
        "Event-Category":"acdc_status_stat",
    }
]
```

  * Thực hiện cuộc gọi: outbound_connecting
```json
{
    "Outbound-Alias":"connecting",
    "Call-ID":"ZDc5ZTA4ZjI0ZjJlOTAwZDMzYTk0NGQyNzExYWY4NTg.",
    "Unix-Time-Ms":1722652471053,
    "Agent-ID":"8b14e6242b49c65f68b85a7f4075a989",
    "Account-ID":"e7339ac33c3b5027922fde17b88eb772",
    "Event-Name":"outbound",
    "Event-Category":"acdc_status_stat",
}
```

  * Thực hiện cuộc gọi: outbound_connected
```json
{
    "Outbound-Alias":"connected",
    "Call-ID":"ZDc5ZTA4ZjI0ZjJlOTAwZDMzYTk0NGQyNzExYWY4NTg.",
    "Unix-Time-Ms":1722652471053,
    "Agent-ID":"8b14e6242b49c65f68b85a7f4075a989",
    "Account-ID":"e7339ac33c3b5027922fde17b88eb772",
    "Event-Name":"outbound",
    "Event-Category":"acdc_status_stat",
}
```

# Resource Flags

## Cấu hình flags của Resource
Ví dụ về cấu hình flags của một Resource
```json
{
    "data": {
        "flags": ["free_charge", "expert_call"],
        "require_flags": true
    }
}
```

Field | Type | Default | Description
--- | --- | --- | ---
require_flags | boolean | false |
flags | [String] | [ ] |

* Khi `require_flags=true` thì cuộc gọi muốn sử dụng được Resource này thì bắt buộc `call outbound flags` phải có giá trị và là tập con của `Resource flags`. Các giá trị được coi là matching:
  * free_charge
  * expert_call
  * free_charge, expert_call
* Khi `require_flags=false` thì cuộc gọi có `call outbound flags = []` vẫn có thể được sử dụng Resource. Nhưng khi `call outbound flags` có giá trị thì nó vẫn phải là tập con của `Resource flags`

## Set Call outbound flags
Để đặt outbound flags cho một cuộc gọi, ta sử dụng qua một trong hai, hoặc cả 2 thuộc tính là `outbound_flags` và `dynamic_flags`
```json
{
    "data": {
        "outbound_flags": ["expert_call"],
        "dynamic_flags": [
            "zone",
            "custom_channel_vars.resource_flag"
        ],
        "use_local_resources": true
    },
    "module": "resources",
    "children": {}
}
```
Field | Type | Default | Description
--- | --- | --- | ---
outbound_flags | [String] | [] | Các giá trị tĩnh (static) sẽ được thêm vào Outbound-Flags của cuộc gọi
dynamic_flags | [String] | [] | Các giá trị động (dymanic) tùy theo giá trị trong biến của cuộc gọi

Nếu cuộc gọi đang ở `zone=z0` và biến `custom_channel_vars.resource_flag=free_charge` thì với cấu hình ở trên cuộc gọi sẽ có Outbound-Flags là:
* z0 | free_charge | expert_call
* => Hệ thống cần tìm một Resource có đủ 3 flags trên thì mới có thể thực hiện cuộc gọi ra

Khi có Resource đáp ứng thì tại ở cả channel inbound và outbound sẽ có biến CCV là:
* "Outbound-Flags" : "free_charge|z0|expert",

## Cài đặt để đánh dấu cuộc gọi tới chuyên gia (expert)

### Callflow
#### Callflow: nhóm chuyên gia
```json
{
    "data": {
        "custom_application_vars": {
            "expert_flag": "expert_free_charge" /* hoặc "expert_default_charge" */
        },
        "export": true
    },
    "module": "set_variables",
    "children": {
        "_": {/* dynamic_cid hoặc ring_group nhóm chuyên gia */}
    }
}
```

#### Callflow: no_match
```json
{
    "data": {
        "flow": {
            "data": {
                "dynamic_flags": ["custom_channel_vars.expert_flag"],
                "use_local_resources": true
            },
            "module": "resources",
            "children": {}
        },
        "numbers": [
            "no_match"
        ],
        "type": "no_match"
    }
}
```

### Resource
#### Trunk hiển thị số khách hàng, nhưng không tính phí với khách hàng
```json
{
    "data": {
        "flags": ["expert_free_charge"],
        "require_flags": true
    }
}
```

#### Trunk gọi ra thông thường (có thể dùng chung với callout thông thường)
```json
{
    "data": {
        "flags": ["expert_default_charge"],
        "require_flags": false
    }
}
```

### CDR
Trên Channel Inbound (khách hàng) và cả Outbound (chuyên gia) của cuộc gọi chuyên gia sẽ có biến `custom_channel_vars.expert_flag` hoặc `custom_channel_vars.Outbound-Flags` có giá trị là `expert_free_charge` hoặc `expert_default_charge`
giúp nhận biết cuộc gọi là dc chuyển tới chuyên gia.
