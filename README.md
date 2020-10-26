# ZohoCreator-Pull-and-Push-with-Subforms
A Deluge script that pulls records from Books/CRM/another database into a Zoho Creator Form as Subform Rows, and then push changes to each individual record (or perform other actions like creating records) in the parent system.

## Core Idea
Suppose you have a long list of records in Books/CRM/another database that needs to be manually reviewed before you decide to execute certain actions. Instead of going through each and every record on the database (which is tideous and inefficient), custom functions can be written to **pull** relevant record information from the database into Creator as Subform Rows. Edits to those subform rows can affect the related record upon submission. Custom actions could also be performed around each record via a Decision Box field on the subform row that would act as a general trigger.

## Example Case 1 (Zoho Books)
To aid the illustration, we will use an example case: Periodically, the accounting team needs to review all overdue invoices from Zoho Books, and decide if they want to levy a finance charge (late fee) on each of the invoices.

## Configuration
* A Creator Form has to be created with a subform containing all necessary fields from Books for reviewing purposes and for the function to work (these can be hidden fields - admin only). In this example, we need the following:
  * Customer Name
  * Invoice No
  * Due Date
  * Outstanding Amount
  * Invoice URL
  * Customer ID (admin only)
  * Invoice ID (admin only)
* Several custom fields can be added into the subform on Creator. This is arbitrary based on your objective. In this example, the following are required:
  * Levy Finance Charge (Decision Box field for accountants to tick to charge the customer).
    * This field acts as a filter for the function execution (on submission, the actions will be executed for every row that has the decision box checked).
  * Interest Charge (Number field to load the interest calculation for Finance Charge which will be set in the script).
  * Current Finance Charge Date (populates the date of the latest finance charge if a charge has been levied on the invoice before).
* Several custom fields need to be created in Books - this depends on what you want to achieve. For this example, we need the following custom fields:
  * Current Finance Charge [Date Field]
    * If a finance charge has been levied on an invoice before, it will print out the date of the finance charge invoice creation. If the original invoice has had 2 charges levied over the course of 2 months, the field will show the latest finance charge date.
  * Finance Charge Invoice(s) [Multi-Line Field]
    * A field that stores every single finance charge invoice no. ever created for the overdue invoice. If an overdue invoice has 4 finance charge invoices, it will print all 4 with line breaks. This will be for the accountant’s reference.
  * FC [Checkbox Field]
    * Allows the script to differentiate between the original invoice vs finance charge invoice.
* A custom item in Books needs to be created for the Finance Charge Invoice Creation.

## Zoho Books Connection
The following connections need to be set up in Creator (Main Dashboard > Account Setup > Extensions > Connections):
* ZohoBooks.invoices.READ
* ZohoBooks.invoices.CREATE
* ZohoBooks.invoices.UPDATE

## PART 1 - PULL
Get all Overdue Invoices from Zoho Books, and pull the necessary fields to a subform in Creator. Here, accountants get to view all overdue invoices in one screen and tick the rows where late fees should be charged. The following script needs to be written **on load** of the Creator form.
* Record Event (Created) > Form Event (Load of the Form)

### Get the Org ID
Org ID refers to your organization ID as specified in Zoho Books. It will be needed for the API calls later.
```javascript
//Get the Org ID
getOrganizations = invokeurl
[
	url :"https://books.zoho.com/api/v3/organizations"
	type :GET
	connection:"zohobooks" // Change this to your connection name
];
orgId = getOrganizations.get("organizations").get(0).get("organization_id");
info orgId;
```

### Get all required Records in a List
A pagination iterator is used to get > 200 records (API limit) - read more about API pagination here: https://github.com/TheWorkflowAcademy/api-pagination-zohocrm. 

```javascript
// List of page numbers. This should start with '1' and be much longer than the expected number of pages.
pageIterationList = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25};
// Number of results to return per page. DO NOT exceed the per_page limit of your API, or this function will not perform correctly.
perPageLimit = 200;
// List to accumulate all records
allRecords = List();
// The 'while' condition for evaluation. While this is 'false', the API requests will continue. 
iterationComplete = false;
// Loop over each page in the page list
for each  page in pageIterationList
{
	// Evaluate whether 'while' condition is satisfied
	if(iterationComplete == false)
	{
		paramap = Map();                                //Change the paramap based on your requirement
		paramap.put("status","overdue");
		paramap.put("cf_fc",false);
		paramap.put("sort_column","customer_name");
		response = invokeurl
[
	url :"https://books.zoho.com/api/v3/invoices?organization_id=" + orgId + "&page=" + page + "&per_page=" + perPageLimit
	type :GET
	parameters:paramap
	connection:"zohobooks"  // Change this to your connection name
];
		// Get records from API response. Add them to allRecords list.
		records = response.get("invoices");
		allRecords.addAll(records);
		// Update 'while' condition status
		if(records.size() < perPageLimit)
		{
			iterationComplete = true;
		}
	}
}
// Check for correctness
info "Number of Records: " + allRecords.size();
```

### Pull required record info to the Creator Subform
For each record, we get the necessary fields from Books, and assign values to the subform fields. When this is completed, you will get all the fields of the records you need populated in the Creator subform upon the load of the form, ready for review. 

```javascript
n = 0;
for each  r in allRecords
{
		// Declaring the row
		row_n = Invoice_Financing.Overdue_Invoices();
		// Assigning values to subform fields in the row
		row_n.Invoice_No=r.get("invoice_number");
		row_n.Invoice_ID=r.get("invoice_id");
		row_n.Customer_Name=r.get("customer_name");
		row_n.Due_Date=r.get("due_date");
		row_n.Outstanding_Amount=r.get("balance");
		row_n.Invoice_URL="https://books.zoho.com/app#/invoices/" + r.get("invoice_id");
		// The .002739 is 1/365, i.e. the rate of interest charged per day
		row_n.Interest_Charge=days360(r.get("due_date").toDate(),today) * 0.18 * r.get("balance") * .002739;  // Arbitrary calculation for interest charge
		row_n.Customer_ID=r.get("customer_id");
		row_n.Current_Finance_Charge_Date=r.get("custom_field_hash").get("cf_next_finance_charge_unformatted");
		// declare a variable to hold the collection of rows
		update = Collection();
		update.insert(row_n);
		// insert the rows into the subform through the variable
		input.Overdue_Invoices.insert(update);
		n = n + 1;
}
```
*Note: Change the row names `row_n` & field API names `r.get("field_name")` based on your requirements.*

## PART 2 - PUSH
When the accounting team is done reviewing the invoices, upon submission of the form, 2 main actions are executed for rows where the "Levy Finance Charge" field is ticked (detailed mechanics and criteria to be explained as we go):
* Create Finance Charge Invoices in Books.
* Information pushed in the Original Invoice in Books.

The following script needs to be written **on submission** of the Creator form.
  * Record Event (Created) > Form Event (Successful Form Submissions)

### Get the Org ID
Org ID refers to your organization ID as specified in Zoho Books. It will be needed for the API calls later.
```javascript
//Get the Org ID
getOrganizations = invokeurl
[
	url :"https://books.zoho.com/api/v3/organizations"
	type :GET
	connection:"zohobooks" // Change this to your connection name
];
orgId = getOrganizations.get("organizations").get(0).get("organization_id");
info orgId;
```

### Print all relevant Customer IDs in a List
For each row with "Levy Finance Charge" ticked, print the customer ID in a list. This will get you a unique list of customer IDs (some customers may have 2 or more overdue invoices) - this is to ensure that only 1 invoice is created per customer. The subsequent part of the script will be iterating through this list.

```javascript
customerIDs = List();
for each  row in Overdue_Invoices
{
	if(row.Levy_Finance_Charge = true)
	{
		if(!customerIDs.contains(row.Customer_ID))
		{
			customerIDs.add(row.Customer_ID);
		}
	}
}
```

### Execute Actions
The script below executes the 2 main actions:
* For each **customer** in the customerID list, a finance charge invoice will be created and marked as sent, with the original invoice number printed in the description of the line item. The charge per line item is based on the Interest Charge field (which was calculated and printed via the onLoad script).
  * If a customer has 5 overdue invoices, it will create 1 **finance charge invoice** with 5 line items, each linked to individual original invoice numbers
* For each **finance charge invoice** created,
  *  The “FC” field is ticked on the finance charge invoice record (this will prevent finance charge invoices from showing up in the Creator form).
* For each **original invoice**, information is updated to the custom fields of the record:
  * The “Current Finance Charge” field will be updated with the finance charge invoice creation date.
  * The “Finance Charge Invoice(s)” field will be updated with the finance charge invoice number(s) linked to it.

```javascript
for each  c in customerIDs
{
	prevFC = "";
	currFC = "";
	line_items = List();
	invoiceIDList = List();
  
	for each  row in Overdue_Invoices
	{
		if(row.Levy_Finance_Charge = true)
		{
			if(c = row.Customer_ID)
			{
				line_item = Map();
				line_item.put("description",row.Invoice_No);
				line_item.put("rate",row.Interest_Charge);
				line_item.put("item_id",2066391000003720076);   // This is a custom item created for Finance Charge. Create your own and replace the ID here.
				line_item.put("quantity",1);
				line_items.add(line_item);
				invoiceIDList.add(row.Invoice_ID);
			}
		}
	}
  
	invoicemap = Map();
	invoicemap.put("customer_id",c);
	invoicemap.put("date",today);
	invoicemap.put("due_date",today.addBusinessDay(10));
	invoicemap.put("line_items",line_items);
  
	// Create List of Custom Fields to Check the FC field
	customFieldList2 = List();
	customFieldMap3 = Map();
	customFieldMap3.put("label","FC");
	customFieldMap3.put("value",true);
	customFieldList2.add(customFieldMap3);
	invoicemap.put("custom_fields",customFieldList2);
  
  // Create the Invoice
	createInvoice = zoho.books.createRecord("Invoices",orgId.toString(),invoicemap);
	info createInvoice;
  
	//Mark created invoice as sent
	marksent = invokeurl
[
	url :"https://books.zoho.com/api/v3/invoices/" + createInvoice.get("invoice").get("invoice_id") + "/status/sent?organization_id=" + orgId
	type :POST
	connection:"zohobooks"  // Change this to your connection name
];
	info marksent;
  
	currentDate = zoho.currentdate.toString("yyyy-MM-dd");
	//info currentDate;
  
	// Create List of Custom Fields for Current Finance Charge
	financeCharge = Map();
	customFieldList = List();
	customFieldMap1 = Map();
	customFieldMap1.put("label","Current Finance Charge");
	customFieldMap1.put("value",currentDate);
	customFieldList.add(customFieldMap1);
	
  for each  row in Overdue_Invoices
	{
		if(row.Levy_Finance_Charge = true)
		{
			if(c = row.Customer_ID)
			{
				// Get current CF Invoice no
				currFC = createInvoice.get("invoice").get("invoice_number");
        
        // Get the original Invoice no
				oriInvoice = zoho.books.getRecordsByID("invoices",orgId,row.Invoice_ID);
        
				// Get previous CF invoice no. if exists
				if(oriInvoice.get("invoice").get("custom_field_hash").get("cf_finance_charge_invoices") != null)
				{
					prevFC = oriInvoice.get("invoice").get("custom_field_hash").get("cf_finance_charge_invoices");
					prevFC = prevFC + "\n";
					fcInvoice = prevFC + currFC;
				}
				else
				{
					fcInvoice = currFC;
				}
				info "oriInvoice" : + oriInvoice;
				info "fcInvoice: " + fcInvoice;
			}
		}
	}
  
	// Create List of Custom Fields for Finance Charge Invoice(s)
	customFieldMap2 = Map();
	customFieldMap2.put("label","Finance Charge Invoice(s)");
	customFieldMap2.put("value",fcInvoice);
	customFieldList.add(customFieldMap2);
	
  // Add all Custom Field Maps into Custom Field List
	financeCharge.put("custom_fields",customFieldList);
	for each  id in invoiceIDList
	{
		updateInvoice = invokeurl
[
	url :"https://books.zoho.com/api/v3/invoices/" + id + "?organization_id=" + orgId
	type :PUT
	parameters:financeCharge + ""
	connection:"zohobooks"  // Change this to your connection name
];
		info updateInvoice;
	}
}
```

## Example Case 2 (CRM)
We have demonstrated how this works with Zoho Books. Now, we will use another example case for Zoho CRM: You are running an online course business and you would like your teachers to leave a remark on certain students which are exceptional (all students are stored in the Contacts module).

## Configuration
* A Creator Form has to be created with a subform containing all necessary fields from CRM for reviewing purposes and for the function to work (these can be hidden fields - admin only). In this example, we need the following:
  * Student Name
  * Student ID (admin only)
* Custom fields can be added into the subform on Creator. This is arbitrary based on your objective. In this example, the following will be added:
  * Update (Decision Box field for teachers to tick for students which they have remarks for).
    * This field acts as a filter for the function execution (on submission, the actions will be executed for every row that has the decision box checked).
  * Teacher's Remark (multi-line field)
    * For teacher's to key in their individual remarks about the students
    * This custom field also needs to be created in CRM.

## Zoho CRM Connection
The following connections need to be set up in Creator (Main Dashboard > Account Setup > Extensions > Connections):
* ZohoCRM.modules.ALL

## PART 1 - PULL
Get all Contacts from Zoho CRM, and pull the necessary fields to a subform in Creator. Here, teachers get to view all students in one screen and leave their remarks. The following script needs to be written **on load** of the Creator form.
* Record Event (Created) > Form Event (Load of the Form)

### Get all required Records in a List
A pagination iterator is used to get > 200 records (API limit) - read more about API pagination here: https://github.com/TheWorkflowAcademy/api-pagination-zohocrm. 

```javascript
// CONFIG: Name of Zoho CRM record module (Contacts, Leads, Custom Modules, etc.)
zohoCrmModule = "Contacts";
// CONFIG: Number of results to return per page. DO NOT exceed the per_page limit of your API, or this function will not perform correctly.
perPageLimit = 200;
// List of page numbers. This should start with '1' and be much longer than the expected number of pages.
pageIterationList = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25};
// List to accumulate all records
allRecords = List();
// The 'while' condition for evaluation. While this is 'false', the API requests will continue. 
iterationComplete = false;
// Loop over each page in the page list
for each  page in pageIterationList
{
	// Evaluate whether 'while' condition is satisfied
	if(iterationComplete == false)
	{
		// Get records from Zoho CRM API.
		// CONFIG: Name of Zoho CRM API Connection
		response = invokeurl
[
	url :"https://www.zohoapis.com/crm/v2/" + zohoCrmModule + "?page=" + page + "&per_page=" + perPageLimit
	type :GET
	connection:"zohocrm" // Change this to your connection name
];
		// Get records from API response. Add them to allRecords list.
		records = response.get("data");
		allRecords.addAll(records);
		// Update 'while' condition status
		if(records.size() < perPageLimit)
		{
			iterationComplete = true;
		}
	}
}
// Check for correctness
info "Number of Records: " + allRecords.size();

```

### Pull required record info to the Creator Subform
For each record, we get the necessary fields from Books, and assign values to the subform fields. When this is completed, you will get all the fields of the records you need populated in the Creator subform upon the load of the form, ready for review. 

```javascript
n = 0;
for each  r in allRecords
{
		// Declaring the row
		row_n = Student_Review_Form.List_of_Students();
		// Assigning values to subform fields in the row
		row_n.Student_Name = r.get("First_Name") + " " + r.get("Last_Name");
		row_n.Student_ID = r.get("id");
		// declare a variable to hold the collection of rows
		update = Collection();
		update.insert(row_n);
		// insert the rows into the subform through the variable
		input.List_of_Students.insert(update);
		n = n + 1;
}
```
*Note: Change the row names `row_n` & field API names `r.get("field_name")` based on your requirements.*

## PART 2 - PUSH
When the teachers are done reviewing the students, upon submission of the form, the function will then **push** the remark and update into CRM. The following script needs to be written **on submission** of the Creator form.
  * Record Event (Created) > Form Event (Successful Form Submissions)

### Push updates to record in CRM
For each student in the subform, if the "Update" Decision Box is ticked, the function will get the remark keyed in by the teacher, and update the similar "Teacher's Remark" field in CRM accordingly.

```javascript
for each  row in List_of_Students
{
	if(row.Update == true)
	{
		//Create map for update
		data = List();
		content = Map();
		content.put("id", row.Student_ID);
		content.put("Teacher_s_Remark",row.Teacher_s_Remark);
		data.add(content);
		mp = Map();
		mp.put("data", data);

		//Update record in CRM
		response = invokeurl
		[
			url :"https://www.zohoapis.com/crm/v2/Contacts"
			type :PUT
			parameters:mp.toString()
			connection:"zohocrm"	// Change this to your connection name
		];
		info response;
	}
}
```
