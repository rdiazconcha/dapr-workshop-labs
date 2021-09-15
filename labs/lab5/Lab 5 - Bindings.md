# Lab 5 - Bindings
In this lab, you will use the input and output binding building blocks for receiving and sending data in the DaprHospital application.

## Task 1: Provision the Azure Storage Account
1. Create a resource group named "daprhospital" in your Azure subscription.

2. Create a new storage account with a unique name.

3. After the Storage account is successfully deployed, obtain its access key and copy it in the clipboard.

## Task 2: Copy the new component files
1. Copy both azure-queue-input-binding.yaml and azure-queue-output-binding.yaml files located in the src\components folder, to the components folder of your .dapr directory ($HOME\\.dapr\components).

2. Modify the azure-queue-input-binding.yaml file:
    a. Set the storageAccount name to the name that you chose.
    b. Set the storageAccessKey to the value that you copied to the notepad.
    c. Queue name should be left as "peoplequeue".
    
3. Modify the azure-queue-output-binding.yaml file:
    a. Set the storageAccount name to the name that you chose.
    b. Set the storageAccessKey to the value that you copied to the notepad.
    c. Queue name should be left as "proceduresqueue".
    
## Task 3: Modify the Person microservice
1. In the PersonController class:
    a. Implement a new method named GetCommandFromRequestAsync() that converts the request body from a Base64 string to a CreatePersonCommand object.
    
    b. Implement a new method named OnInputBinding().  Decorate it with the [HttpPost("/azurequeueinput")] attribute.  Invoke the GetCommandFromRequestAsync() method to get the deserialized command.
    
        ```csharp
        [HttpPost("/azurequeueinput")]
        public async Task<IActionResult> OnInputBinding()
        {
            var command = await GetCommandFromRequestAsync();
            var person = await CreatePersonAndPublishEventAsync(command);
            return Ok(person.Id);
        }
        
        private async Task<CreatePersonCommand> GetCommandFromRequestAsync()
        {
            using var streamReader = new StreamReader(Request.Body);
            var body = await streamReader.ReadToEndAsync();
            var bytes = Convert.FromBase64String(body);
            var decodedString = System.Text.Encoding.UTF8.GetString(bytes);
            var command = JsonConvert.DeserializeObject<CreatePersonCommand>(decodedString);
            return command;
        }
        ```

## Task 4: Modify the Hospital microservice
1. In the HospitalController class:
    a. Inject a DaprClient object in the constructor.
    b. Implement a new method named AddProcedure() that takes an AddProcedureCommand object, and performs the following:
        - Add a new procedure to the Inpatient.
        - Save the changes to the database by using the DbContext.
        - Invoke the "azurequeueoutput" binding by using the DaprClient object instance that was injected in the constructor.
        ```csharp
        [ApiController]
        [Route("[controller]")]
        public class HospitalController : ControllerBase
        {
            private readonly ILogger<HospitalController> logger;
            private readonly HospitalDbContext dbContext;
            private readonly DaprClient daprClient;
        
            public HospitalController(ILogger<HospitalController> logger, HospitalDbContext dbContext, DaprClient daprClient)
            {
                this.logger = logger;
                this.dbContext = dbContext;
                this.daprClient = daprClient;
            }
        
            [HttpPost("procedure")]
            public async Task<IActionResult> AddProcedure(AddProcedureCommand command)
            {
                var inpatientToAddProcedure = await dbContext.Inpatients.Include(p => p.Procedures).FirstAsync(i => i.Id == command.PatientId);
                if (inpatientToAddProcedure == null)
                {
                    return NotFound();
                }
                var newProcedure = new Procedure(command.ProcedureName);
                inpatientToAddProcedure.AddProcedure(newProcedure);
                dbContext.Inpatients.Update(inpatientToAddProcedure);
                var message = $"Adding procedure {command.ProcedureName} to patient with Id: {command.PatientId}";
                logger?.LogInformation(message);
                await dbContext.SaveChangesAsync();
        
                await daprClient.InvokeBindingAsync("azurequeueoutput", "create", message);
        
                return Ok();
            }
        }
        ```
## Task 5: Test the bindings
1. Execute all services by using `dapr run` or `tye run`.
2. Create a new message in the "peoplequeue" queue by using the Azure Portal with the following payload:
    ```json
    {
      "firstname" : "FIRST",
      "lastname" : "LAST"
    }
    ```

3. Verify that the new Person was created in the database.
4. Send a POST request to /hospital/procedure to add a new procedure to a patient, and test the output binding.  
5. Verify that there is a new message in the "proceduresqueue" queue.