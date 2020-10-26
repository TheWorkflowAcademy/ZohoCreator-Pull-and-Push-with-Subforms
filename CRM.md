# ZohoCreator-Pull-and-Push-with-Subforms
A Deluge script that pulls records from Books/ CRM other database into a Creator Form as Subform Rows, and then push changes to each individual record (or perform other actions like creating records) in the parent system.

## Core Idea
Suppose you have a long list of records in Books/ CRM/ other database that needs to be manually reviewed before you decide to execute an action. Instead of going through each and every record on the database (which is tideous and inefficient), a custom function can be written to **pull** relevant record information from the database into Creator as Subform Rows. A Decision Box field will be set as a trigger for each row on the subform to **push** the necessary actions upon form submission when ticked. 

## Example Case 2 (CRM)
We have demonstrated how this works with Zoho Books in the main branch. Now, we will use another example case for Zoho CRM: You are running an online course business and you would like your teachers to leave a remark on certain students which are exceptional (all students are stored in the Contacts module).

## Configuration
* A Creator Form has to be created with a subform containing all necessary fields you would want to pull from Books for reviewing purposes and for the function to work (these can be hidden fields - admin only). In this example, we need the following:
  * Student Name
  * Student ID (admin only)
* Custom fields can be added into the subform on Creator. This is arbitrary based on your objective. In this example, the following will be added:
  * Levy Finance Charge (Decision Box field for teachers to tick for students which they have remarks for).
    * This field acts as a filter for the function execution (on submission, the actions will be executed for every row that has the decision box checked).
  * Teacher's Remark (multi-line field)
    * For teacher's to key in their individual remarks about the students
    * This custom field also needs to be created in CRM.

## Zoho CRM Connection
The following connections need to be set up in Creator (Main Dashboard > Account Setup > Extensions > Connections):
* ZohoCRM.modules.ALL

## PART 1 - PULL
Get all Contacts from Zoho CRM, and pull the necessary fields to a subform in Creator. Here, teachers get to view all students in one screen and leave their remarks. The following script needs to be written on load of the Creator form.
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
For each record, we get the necessary fields from Books, and assign values to the subform fields. Change the row names `row_n` & field API names `r.get("field_name")` based on your own requirements. When this is completed, you will get all the fields of the records you need populated in the Creator subform upon the load of the form, ready for review. 

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

## PART 2 - PUSH
When the teachers are done reviewing the students, upon submission of the form, the function will then **push** the remark and update into CRM. The following script needs to be written on submission of the Creator form.
  * Record Event (Created) > Form Event (Successful Form Submissions)

### Push updates to record in CRM
For each student in the subform, if the Update Decision Box is ticked, the function will get the remark keyed in by the teacher, and populate the similar "Teacher's Remark" field in CRM accordingly.

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
