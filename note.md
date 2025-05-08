### **PTP Configuration Workflow Notes**


Ask what triggers the step function

#### **1. Data Lambda: Understanding and Access**

- Goal: Walk through how to access data from the data lambda.
- Scope: Includes codebase navigation and using the dev console.
- Objective: Understand how to pull contract data.


#### **2. Data Flow Logic**

- Pull **contract** from data lambda.
- **Check:** `contract.status === "enrolled"`
- If enrolled:

This process can be a lambda or something else ask bobby what he is doing around his data after getting it from data lambda from contracts into his step function
    
- Extract `draftScheduledPayments` from the contract:
	
	- This will be a list of scheduled contract payments.
		
	- Key fields: `paymentDate`, `paymentAmount`
            
- Use this list to build input for the **RT Payment Scheduler Lambda**.
    


#### **3. Integration with RT Payment Scheduler**
Create new list with this data that has to include the stuff below and what ever else stuff 
- **Input Needed:**  
    this a list of ptp this is not the exact full input:
    
    ```json
    { 
      paymentDate: string, 
      paymentAmount: number 
    }
    ```

ending output could maybe include these two fields since things aren't finalized


- **Step Function Responsibilities:**
    
    - Send the input to the RT Payment Scheduler.
    - Handle errors per payment request.
    - Await response with status fields.






