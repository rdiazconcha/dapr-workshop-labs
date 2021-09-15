# Lab 4 - State management
In this lab, you will use the State management building block of Dapr, to save state in the default component (Redis).

## Task 1: Modify the PatientQuery microservice
1. In the PatientQueryController class, modify the GetAll() method to save the last query by using GetStateEntryAsync():

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
            var lastQuery = await daprClient.GetStateEntryAsync<StateModel>("statestore", "lastquery");
            if (lastQuery.Value != null && DateTime.UtcNow <= lastQuery.Value.LastQuery.AddSeconds(30))
            {
                return Ok(lastQuery.Value.Data);
            }
    
            IEnumerable<QueryModel> result;
            result = await QueryPatients();
    
            bool saved = false;
            while (!saved)
            {
                result = await QueryPatients();
                lastQuery.Value = new StateModel()
                {
                    LastQuery = DateTime.UtcNow,
                    Data = result
                };
                saved = await lastQuery.TrySaveAsync();
            }
    
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

## Task 2: Test
1. Test by executing all services using `dapr run` or `tye run`.

2. Send a GET request to the /patientquery endpoint.

3. Verify that the state was saved.  You can use Redis Desktop Manager, Redis Commander, or any other Redis exploration tool.