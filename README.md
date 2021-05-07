using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

using Amazon.Lambda.Core;
using System.Net.Http;
using Newtonsoft.Json;
using Amazon.Lambda.APIGatewayEvents;
using System.Dynamic;
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.DocumentModel;
using Amazon.DynamoDBv2.Model;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace govSpendingConnect
{

    public class Function
    {
        //create a client to facilitate HTTP requests
        HttpClient webRequest = new HttpClient();

        //create new connection to dynamo db
        private static AmazonDynamoDBClient client = new AmazonDynamoDBClient();
        private string tableName = "StateSpending";



        public async Task<List<object>> FunctionHandler(APIGatewayProxyRequest input, ILambdaContext context)
        {
            //place items from list into dynamo db table
            Table table = Table.LoadTable(client, tableName);

            //gather uRL to connect to
            var url = "https://api.usaspending.gov/api/v2/recipient/state/";


            //Query the url and get string response
            string response = await webRequest.GetStringAsync(url);


            //add text to string response to allow deserialization
            response = "{stateSpending:" + response + "}";


            //convert results into an object
            dynamic expando = JsonConvert.DeserializeObject<ExpandoObject>(response);


            //create a list of objects
            List<object> listings = new List<object>();

            //Reduce expando object into manageable data
            foreach (object x in expando.stateSpending)
            {
                listings.Add(x);

            }

            //TESTING
            /*
            List<StateListing> stateListing = new List<StateListing>();
            foreach (object x in expando.stateSpending)
            {
                StateListing s = x as StateListing;
                stateListing.Add(s);
            }
            */

            //convert list data into dictionary values
            //Dictionary<string, object> spendingDict = listings.ToDictionary(x => x);


            //-AWS Convention below.  If it's not present, document will return NULL below
            //-----------------------------------------------
            PutItemOperationConfig config = new PutItemOperationConfig();
            config.ReturnValues = ReturnValues.AllOldAttributes;

            //submit items into the Dynamo Table
            foreach (object x in listings)
            {
                await table.PutItemAsync(Document.FromJson(JsonConvert.SerializeObject(x)));
            }


            return listings;
        }
    }
}
