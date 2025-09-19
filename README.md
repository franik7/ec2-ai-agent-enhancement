# EC2 AI-Enhanced Monitoring and Remediation System – ServiceNow Implementation

## System Overview
This project extends the **EC2 Monitoring and Remediation System** (ec2-remediation-system) in ServiceNow with a conversational **AI Agent**.  

Netflix’s DevOps team now has **two remediation paths**:

- **Manual:** Engineers navigate to an EC2 Instance record and use the **Trigger EC2 Remediation** UI Action.  
- **AI-Assisted:** Engineers chat with the **EC2 Remediation Assistant**, which:
  - Accepts either an **AWS `instance_id`** or an **Incident number**  
  - Retrieves the target EC2 record  
  - Explains the intended action and requests **explicit human approval**  
  - Calls the same **RemediationHelper logic** and **AWS Integration Server API** as the manual flow  
  - Logs every attempt to the **Remediation Log table** for auditability  
  - Provides default error handling if the input is malformed or the record cannot be found  

![Scope Screenshot](assets/initial_setup.png)

---

## Implementation Steps
Key steps to integrate the AI Agent into the existing manual remediation system:

1. **Custom Tables (unchanged)**  
   - `EC2 Instance` – instance metadata + status (ON/OFF).  
   - `Remediation Log` – every remediation attempt, payload, response.  
   - `EC2 Monitoring Incidents` – incidents created when status = OFF.  

2. **Manual UI Action (existing)**  
   - *Trigger EC2 Remediation* button on EC2 Instance records.  
   - Calls `EC2RemediationHelper` Script Include with `sys_id`.  

3. **AI Agent Creation**  
   - Agent: *EC2 Remediation Assistant*  
   - Instructions: parse natural language, extract `instance_id` or incident number, confirm steps with user, escalate errors if invalid  
   - Mode: **Supervised** for human approval  

4. **Script Tool (new)**  
   - Input: `instance_id`  
   - Logic: query EC2 Instance table by `instance_id`, call **RemediationHelper** via Connection & Credential Alias, write Remediation Log  
   - Handles: invalid IDs, duplicate matches, missing connection alias  

5. **Agent Integration**  
   - Linked the Script Tool to the AI Agent  
   - Confirmed outputs are structured (booleans, http_status, log_id) for conversational display  

![Script Tool Screenshot](assets/script_tool.png)

---

## Architecture Diagram
Enhanced workflow diagram showing both manual and AI-driven paths:

![Architecture Diagram](Diagram.png)

- **Manual Path**  
  Engineer → EC2 Record → UI Action → Script Include → AWS API → Remediation Log → Flow resolves incident → Slack update  

- **AI Path**  
  Engineer → Chat with Agent → Parse input (instance_id or incident) → Confirm action → Script Tool → Script Include → AWS API → Remediation Log → Flow resolves incident → Slack update  

---

## Optimization
**Manual vs AI Agent comparison**

- **Manual**  
  - Input: `sys_id` (direct record reference)  
  - Strength: fast when engineer is already in ServiceNow console  
  - Limitation: requires navigating forms and knowing the exact record  

- **AI Agent**  
  - Input: AWS `instance_id` or Incident number  
  - Strength: conversational, faster during high-volume incidents, reduces clicks, keeps engineers in chat context  
  - Adds safety: human approval before any remediation  
  - Error handling: clear default responses for invalid formats (e.g., `i-xxxx…`) or unrecognized incident IDs  

![Comparison Screenshot](assets/compare.png)

---

## DevOps Usage
**Two options for Netflix engineers**

1. **Manual Remediation**  
   - Open EC2 Instance record  
   - Click **Trigger EC2 Remediation**  
   - Watch Remediation Log for results; Flow resolves related incident and posts Slack notification  

2. **AI Conversational Remediation**  
   - In AI Agent Studio or chat window:  
     - Example A: `Restart instance i-09ae69f1cb71f622e`  
     - Example B: `Help me with incident EC2_INC001234`  
   - Agent confirms: *“I found instance i-09ae… linked to that record. Do you want me to restart it now?”*  
   - Engineer approves  
   - Agent executes remediation, logs attempt, and provides output details (log_id, http_status)  
   - Flow and Slack notifications update as with manual remediation  

![Agent Conversation Screenshot](assets/agent_convo.png)
