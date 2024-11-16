# How to Create partial view for customers and pass model inside partial view
## Step 1: Design partial view

```csharp
<table cellpadding="0" cellspacing="0" class="grid" id="CustomerGrid">
    <tr>
        <th>Contact Name</th>
        <th>Address</th>
        <th>Mobile</th>
        <th>Email</th>
        <th>City</th>
        <th>GST No.</th>
    </tr>
    @foreach (Customer customer in Model)
    {
        <tr>
            <td>@customer.ContactName</td>
            <td>@customer.Address</td>
            <td>@customer.Mobile</td>
            <td>@customer.Email</td>
            <td>@customer.City</td>
            <td>@customer.GST No.</td>
            <td><a class="details" href="javascript:;">View</a></td>//this will show more detials as model pop-up for customer
        </tr>
    }
</table>
```

## Step 2: Design Customer list page
```csharp
@model IEnumerable<Customer>
<h4>Customers</h4>
<hr />
//here partial view is rendered and model passed inside partial view
<partial name="_CustomerList" model="Model" />
<div id="partialModal" class="modal" tabindex="-1" role="dialog">
    <div class="modal-dialog" role="document">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title">Customer Details Form</h5>
                <button type="button" class="close" data-dismiss="modal" aria-label="Close">
                    <span aria-hidden="true">&times;</span>
                </button>
            </div>
            <div class="modal-body">
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
            </div>
        </div>
    </div>
</div>
```
- now we will use ajax to show detail view in model pop-up. for that add following scripts
```csharp
<script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.8.3/jquery.min.js"></script>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous" />
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js" integrity="sha384-JZR6Spejh4U02d8jOt6vLEHfe/JQGiRRSQQxSfFWpi1MquVdAyjUar5+76PVCmYl" crossorigin="anonymous"></script>
```
- Next is ajax method call
```csharp
<script type="text/javascript">
    $(function () {
        $("#CustomerGrid .details").click(function () {
            var customerId = $(this).closest("tr").find("td").eq(0).html();
            $.ajax({
                type: "POST",
                url: "/Demo/Details",
                data: { "customerId": customerId },
                success: function (response) {
                    $("#partialModal").find(".modal-body").html(response);
                    $("#partialModal").modal('show');
                },
                failure: function (response) {
                    alert(response.responseText);
                },
                error: function (response) {
                    alert(response.responseText);
                }
            });
        });
    });
</script>
```
