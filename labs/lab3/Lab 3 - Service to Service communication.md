# Lab 3 - Service to Service communication

In this lab, you will use the Service invocation building block of Dapr, to synchronously communicate between services in the DaprHospital application.

## Task 1: Modify the Person microservice

1. In the PersonController class, implement the GetAll() method:
    ```csharp
    [HttpGet("all")]
    public async Task<IActionResult> GetAll()
    {
        var all = await dbContext.People.ToListAsync();
        var result = all.Select(p => new PersonModel(p.Id, p.Name.FullName));
        return Ok(result);
    }
    ```

## Task 2: Modify the Patient microservice
1. In the PatientController class, implement the GetAll() method:

    ```csharp
    [HttpGet("all")]
    public async Task<IActionResult> GetAll()
    {
        var all = await dbContext.Patients.ToListAsync();
        var result = all.Select(p => new PatientModel(p.Id, p.BloodType, Enum.GetName(typeof(PatientStatus), p.Status)));
        return Ok(result);
    }
    ```

## Task 3: Modify the PatientQuery microservice
1. Add the Dapr.AspNetCore NuGet package reference.

2. In the Startup class, chain the `AddDapr()` call after `AddControllers()`.

3. In the PatientController class:
    a. Inject DaprClient.
    
    b. Implement the QueryPatients() private method.  Inside this method, use the daprClient.InvokeMethodAsync() method to invoke both the Person and Patient microservices.
    
    c. Implement the GetAll() method.
        - Invoke QueryPatients() and return Ok()
        ```csharp
        [ApiController]
        [Route("[controller]")]
        public class PatientQueryController : ControllerBase
        {
            private readonly DaprClient daprClient;
        
            public PatientQueryController(DaprClient daprClient)
            {
                this.daprClient = daprClient;
            }
        
            [HttpGet]
            public async Task<IActionResult> GetAll()
            {
                var result = await QueryPatients();
                return Ok(result);
            }
        
            private async Task<IEnumerable<QueryModel>> QueryPatients()
            {
                var patients = await daprClient.InvokeMethodAsync<IEnumerable<PatientModel>>(HttpMethod.Get, "patient", "patient/all");
                var people = await daprClient.InvokeMethodAsync<IEnumerable<PersonModel>>(HttpMethod.Get, "person", "person/all");
                var result = from patient in patients
                                join person in people on patient.Id equals person.Id
                                select new QueryModel(patient.Id, person.FullName, patient.BloodType, patient.Status);
                return result;
            }
        }
        ```

## Task 4: Test
1. Test by executing each microservice by using `dapr run` as follows (each one in a Windows Terminal tab or instance):

```powershell
dapr run --app-id person --app-port 5000 --dapr-http-port 50000  -- dotnet run -p .\DaprHospital.Person.Api\
dapr run --app-id patient --app-port 5001 --dapr-http-port 50001  -- dotnet run -p .\DaprHospital.Patient.Api\ --urls http://localhost:5001
dapr run --app-id patientquery --app-port 5002 --dapr-http-port 50002  -- dotnet run -p .\DaprHospital.PatientQuery.Api\ --urls http://localhost:5002
```
2. Send a GET request to /patientquery to obtain the results.