# Lab 2 - Pub/Sub

In this lab, you will use the Pub/Sub building block of Dapr to send and receive messages asynchronously from and to the different microservices in the DaprHospital application.

## Task 1: Modify the Person microservice

1. Add a reference to the `Dapr.AspNetCore` NuGet package.

2. In the Startup class, chain the `AddDapr()` call after `AddControllers()`.

3. In the PersonController class:

    a. Inject `DaprClient` in the constructor and store it in a `daprClient` field
    b. Implement the CreatePersonAndPublishEventAsync() method as follows:
        ```csharp
        private async Task<Domain.Entities.Person> CreatePersonAndPublishEventAsync(CreatePersonCommand command)
        {
            var person = new Domain.Entities.Person();
            person.SetName(PersonName.Create(command.FirstName, command.LastName));
            dbContext.People.Add(person);
            await dbContext.SaveChangesAsync();
        
            var personCreated = new PersonCreatedIntegrationEvent(person.Id);
            logger?.LogInformation($"Sending message: {personCreated}");
            await daprClient.PublishEventAsync("pubsub", "person-topic", personCreated);
            return person;
        }
        ```
    c. Implement the Post() method as follows:
        ```csharp
        [HttpPost]
        public async Task<IActionResult> Post(CreatePersonCommand command)
        {
            var person = await CreatePersonAndPublishEventAsync(command);
            return Ok(person.Id);
        }
        ```

## Task 2: Modify the Patient microservice

1. Add a reference to the `Dapr.AspNetCore` NuGet package.

2. In the Startup class:
    a. Chain the `AddDapr()` call after `AddControllers()`.
    
    b. Invoke `app.UseCloudEvents()` before `app.UseEndpoints()`
    
    c. Invoke `endpoints.MapSubscribeHandler()` before `endpoints.MapControllers()`
    
3. Add a new IntegrationEventsController class:
    a. Create the IntegrationEventsController.cs class inside the Controllers folder
    b. Inject ILogger<IntegrationEventsController>
    c. Inject PatientDbContext
    d. Implement the OnPersonCreated() method.  Decorate it with the [Topic] attribute
    e. Use the “person-topic” topic
        ```csharp
           [ApiController]
           [Route("[controller]")]
           public class IntegrationEventsController : ControllerBase
           {
               private readonly ILogger<IntegrationEventsController> logger;
               private readonly PatientDbContext dbContext;
        
               public IntegrationEventsController(ILogger<IntegrationEventsController> logger, PatientDbContext dbContext)
               {
                   this.logger = logger;
                   this.dbContext = dbContext;
               }
        
               [Topic("pubsub", "person-topic")]
               public async Task<IActionResult> OnPersonCreated(PersonCreatedIntegrationEvent personCreated)
               {
                   logger?.LogInformation($"Message received: {personCreated}");
                   var patient = new Domain.Entities.Patient(personCreated.Id);
                   dbContext.Patients.Add(patient);
                   await dbContext.SaveChangesAsync();
                   return Ok(personCreated.Id);
               }
           }
        ```
4. In the PatientController class:
    a. Inject DaprClient
    b. Implement the AdmitPatient(), DischargePatient(), and SetBloodType() methods
        ```csharp
           [ApiController]
           [Route("[controller]")]
           public class PatientController : ControllerBase
           {
               private readonly ILogger<PatientController> logger;
               private readonly PatientDbContext dbContext;
               private readonly DaprClient daprClient;
        
               public PatientController(ILogger<PatientController> logger, PatientDbContext dbContext, DaprClient daprClient)
               {
                   this.logger = logger;
                   this.dbContext = dbContext;
                   this.daprClient = daprClient;
               }
        
               [HttpPost("admit")]
               public async Task<IActionResult> AdmitPatient(AdmitPatientCommand command)
               {
                   var patientToAdmit = await dbContext.Patients.FindAsync(command.Id);
                   if (patientToAdmit == null)
                   {
                       return NotFound();
                   }
                   patientToAdmit.Admit();
                   dbContext.Patients.Update(patientToAdmit);
                   logger?.LogInformation($"Admitting patient: {command.Id}");
                   await dbContext.SaveChangesAsync();
                   var patientAdmitted = new PatientAdmittedIntegrationEvent(command.Id);
                   await daprClient.PublishEventAsync("pubsub", "patient-topic", patientAdmitted);
        
                   return Ok();
               }
        
               [HttpPost("discharge")]
               public async Task<IActionResult> DischargePatient(DischargePatientCommand command)
               {
                   var patientToDischarge = await dbContext.Patients.FindAsync(command.Id);
                   if (patientToDischarge == null)
                   {
                       return NotFound();
                   }
                   patientToDischarge.Discharge();
                   dbContext.Patients.Update(patientToDischarge);
                   logger?.LogInformation($"Discharging patient: {command.Id}");
                   await dbContext.SaveChangesAsync();
                   var patientDischarged = new PatientDischargedIntegrationEvent(command.Id);
                   await daprClient.PublishEventAsync("pubsub", "patient-topic", patientDischarged);
        
                   return Ok();
               }
        
               [HttpPost("bloodtype")]
               public async Task<IActionResult> SetBloodType(SetBloodTypeCommand command)
               {
                   var patientToSetBloodType = await dbContext.Patients.FindAsync(command.Id);
                   if (patientToSetBloodType == null)
                   {
                       return NotFound();
                   }
                   var bloodType = PatientBloodType.Create(command.BloodType);
                   patientToSetBloodType.SetBloodType(bloodType);
                   dbContext.Patients.Update(patientToSetBloodType);
                   logger?.LogInformation($"Setting the patient {command.Id} blood type to: {command.BloodType}");
                   await dbContext.SaveChangesAsync();
        
                   return Ok();
               }
           }
        ```

## Task 3: Modify the Hospital microservice

1. Add a reference to the `Dapr.AspNetCore` NuGet package.

2. In the Startup class:
    a. Chain the `AddDapr()` call after `AddControllers()`.
    
    b. Invoke `app.UseCloudEvents()` before `app.UseEndpoints()`
    
    c. Invoke `endpoints.MapSubscribeHandler()` before `endpoints.MapControllers()`

3. Add a new IntegrationEventsController class:

    a. Create the `IntegrationEventsController.cs` class file inside the `Controllers` folder
    
    b. Inject `ILogger<IntegrationEventsController>`
    
    c. Inject `PatientDbContext`
    
    d. Implement the OnPatientAdmitted() method.  Decorate it with the [Topic] attribute
    
    e. Use the “patient-topic” topic

        ```csharp
        [ApiController]
        [Route("[controller]")]
        public class IntegrationEventsController : ControllerBase
        {
            private readonly ILogger<IntegrationEventsController> logger;
            private readonly HospitalDbContext dbContext;
        
            public IntegrationEventsController(ILogger<IntegrationEventsController> logger,
                                                HospitalDbContext dbContext)
            {
                this.logger = logger;
                this.dbContext = dbContext;
            }
        
            [Topic("pubsub", "patient-topic")]
            public async Task<IActionResult> OnPatientAdmitted(PatientAdmittedIntegrationEvent @event)
            {
                var existingInpatient = dbContext.Inpatients.FirstOrDefault(i => i.Id == @event.PatientId);
                if (existingInpatient != null)
                {
                    return NoContent();
                }
                var newInpatient = new Inpatient(@event.PatientId);
                dbContext.Inpatients.Add(newInpatient);
                logger?.LogInformation($"Adding inpatient: {@event.PatientId}");
                await dbContext.SaveChangesAsync();
        
                return Ok();
            }
        }
        ```
## Task 4: Test the Pub/Sub functionality
1. Execute all the microservices by using `dapr run`.
2. Send a POST request to the Person microservice (/person) with the following payload:
    ```json
    {
      "firstname" : "FIRST",
      "lastname" : "LAST"
    }
    ```
3. Verify that both the Person and Patient records were created.

4. Send a POST request to the admit endpoint in the Patient microservice (/patient/admit) with the following payload:
    ```json
    {
      "Id" : "PATIENT_GUID"
    }
    ```
    ...where "PATIENT_GUID" is the identifier of the recently created Patient.
    
5. Verify that the Inpatient record was created.

