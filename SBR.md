Định tuyến theo kỹ năng của agent  SBR  
## Mô tả  
* Agent:  
  * Agent có thể có nhiều kỹ năng khác & mức độ thành thạo nhau, ví dụ như: **English**, **Level 5**  
  * Kỹ năng có thể được cấu hình để gắn với một số queues, hoặc cho tất cả queue  
  
  
* Khách hàng (member_call): Trước khi khách hàng vào queue, có thể set biến để yêu cầu về kỹ năng của agent cần có khi trả lời cuộc gọi  
  * Mandatory Skills: yêu cầu agent phải có kỹ năng bắt buộc này mới có thể được phân bổ cuộc gọi  
  * Optional Skills: agent có thể có hoặc không có kỹ năng này, agent nào có thì tổng điểm kỹ năng sẽ cao hơn do đó khả năng được phân bổ cũng sẽ cao hơn  
* Queue:  
  * Queue có thể được cấu hình để enable/disable việc chỉ phân bổ cuộc gọi tới agent có kỹ năng phù hợp và ưu tiên agent  
  có tổng điểm kỹ năng cao hơn khi phân bổ cuộc gọi  
  * Giả sử Queue có bật phân bổ theo kỹ năng (SBR), thì SBR sẽ hoạt động kết hợp với luật phân bổ cũ là round_robin  
  hoặc most_idle & last called agent:  
    * Nếu queue có bật `last_called_agent`, agent đang ready, trước đó cũng đã gặp khách hàng, tại thời điểm của cuộc gọi mới:  
	      * Nếu Agent không có kỹ năng phù hợp với cuộc gọi mới thì agent không được phân bổ cuộc gọi này  
	      * Nếu có kỹ năng phù hợp, thì dù điểm kỹ năng có như nào sẽ được ưu tiên phân bổ  
    * Khách hàng vào queue mà không yêu cầu kỹ năng agent => chọn agent theo round_robin hoặc most_idle  
    * Khách hàng có yêu cầu kỹ năng, nhiều agent thỏa mãn và cùng có tổng điểm kỹ năng cao nhất => chỉ chọn trong list agent có điểm cao nhất này nhưng là agent sẽ tới lượt gần nhất theo luật round_robin hoặc most_idle  
    * Khách hàng có yêu cầu mandatory skill mà không có agent nào thỏa mãn => tiếp tục chờ trong queue  
  
## use cases  
* **Một queue phục vụ nhiều kỹ năng khác nhau**  
  * Queue phục vụ chung cả tiếng việt và tiếng hàn, cần quản lý profile khách hàng để biết yêu cầu về ngôn ngữ  
  trước khi vào queue  
  
* **Test Agent**  
  * Đặt skill cho agent, ví dụ: `answer_test_call_1`  
  * Cuộc gọi test của giám sát, trước khi vào queue thì yêu cầu kỹ năng bắt buộc là: `answer_test_call_1` để chỉ agent nào có kỹ năng này mới được phân bổ cuộc gọi  
  * có thể tạo nhiều nhóm test khác nhau qua skill name  
  
* **Ưu tiên Agent trả lời cuộc gọi**  
  * Đặt skill cho các agent muốn ưu tiên hơn, ví dụ skill: `agent_phuc_vu_chinh`, các agent không ưu tiên (backup) thì không có skill này  
  * Cài đặt cf_acdc_member, để đặt `member_optional_skills=agent_phuc_vu_chinh`, khi đó agent nào có skill này sẽ được ưu tiên phân bổ  
  
## Cấu hình  
  
### Đặt kỹ năng của Agent  
  
```shell  
curl -v -X PATCH \-H "Content-Type: application/json" \  
-H "X-Auth-Token: {AUTH_TOKEN}" \  
-d '  
{  
    "data": {        
	    "skills": {            
		    "english": {                
			    "level": 3            
			},
			"test" : {
				"queues": ["c0d1c2c33e92e9ebd3189cca4cddc88c"]            
			}        
		}    
	}
}' http://{SERVER}:8000/v2/accounts/{ACCOUNT_ID}/users/{AGENT_ID}  
```  
  
| Field   | Type     | Default | Description                                                                                               |
| ------- | -------- | ------- | --------------------------------------------------------------------------------------------------------- |
| skills  | Json     | null    | đầu số của nhóm queue chứa queue hiện tại                                                                 |
| english | String   | null    | mã kỹ năng, ví dụ trên là english, test                                                                   |
| level   | Interger | 1       | mức độ thành thạo của kỹ năng, cao hơn là thành thạo hơn                                                  |
| queues  | List     | []      | kỹ năng này là được chỉ định cho queue nào, nếu queues là [] thì kỹ năng này được dùng ở tất cả các queue |

### Cài đặt queue  
  
* skill_based_routing: (xem tại: `Queue Setting`) để bật/tắt việc phân bổ theo kỹ năng của queue  
  
### Cài đặt cuộc gọi  
* set_variables: `member_optional_skills` hoặc `member_mandatory_skills` hoặc cả 2 nếu cần:  
```json  
{  
    "data": {        
	    "custom_application_vars": {            
	    "member_optional_skills": "o_skill1, o_skill2, o_skill3",           
	    "member_mandatory_skills": "m_skill1, m_skill2, m_skill3"        
	    }    
	},    
	"module": "set_variables",    
	"children": {        
		"_": {FLOW_TO_CF_ACDC_MEMBER}    
	}
}
```  
  
* cf_acdc_member: Đặt `member_optional_skills` hoặc `member_mandatory_skills` hoặc cả 2 nếu cần ở node cf_acdc_member nếu muốn set cho tất cả cuộc gọi  
  * note: member_optional_skills, member_mandatory_skills ở CAVs sẽ được ưu tiên cao hơn ở cf_acdc_member
