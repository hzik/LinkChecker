# Link checker

This custom element gets all links from all rich text elements on your item and checks whether they are live or not:

![screenshot](http://amend.cz/link_checker.gif)

## Configuration

```json
{
    "request_repeater": "https://2jp66x1y922.execute-api.eu-central-1.amazonaws.com/default/requestRepeater"
}
```

You need to link your request repeater in the `request_repeater` parameter.

To set up `request_repeater` above, please follow [Working with sensitive data in custom elements](https://docs.kontent.ai/tutorials/develop-apps/integrate/working-with-sensitive-data-in-custom-elements).
In **Step 2: Configuring your Lambda function**, use the following keys and values in the **Environment variables** section:
  - `BEARER_TOKEN`: `<Preview Delivery API key>`
  - `HOST`: `preview-deliver.kontent.ai`
  - `PATH`: `/<Project ID>/items`
  
Adjust the code of this lambda function to:

```javascript
const https = require("https");
const querystring = require("querystring");

/* ========Config Section======== */
const host = process.env.HOST;
const path = process.env.PATH;
const accessControlAllowOriginValue = process.env.ACCESS_CONTROL_ALLOW_ORIGIN;
const accessControlAllowHeadersValue = process.env.ACCESS_CONTROL_ALLOW_HEADERS;

// Bearer token authentization
const bearerToken = process.env.BEARER_TOKEN;

//  Basic authentication credentials   
const username = process.env.USERNAME;
const password = process.env.PASSWORD;
/* ========Config Section======== */

let authorizationHeaderValue;

if (bearerToken || (username && password)) {
    authorizationHeaderValue = bearerToken ?
        `Bearer ${bearerToken}` :
        `Basic ${new Buffer(username + ":" + password).toString("base64")}`;
}


let domain = '';
let endpoint = '/';
let external = false;

const request = (queryStringParameters, headers, body) => {

    if (queryStringParameters) {
        if (queryStringParameters.external=="yes") {
            external = true;
            let externalurl = JSON.parse(body).url;
            domain = externalurl.split('/')[2];
            if (externalurl.indexOf("/", 10)>0) {
                endpoint = externalurl.substr(externalurl.indexOf("/", 10));
            }
            else {
                endpoint = '';
            }
        }
        else {
            external = false;
            domain = host;
            endpoint = `${path}?${querystring.stringify(queryStringParameters)}`;
        }
    }

    if (authorizationHeaderValue) {
        headers["Authorization"] = authorizationHeaderValue;
    }

    headers["Accept"] = "*/*";
    headers["accept-encoding"] = "identity";
    headers["Host"] = domain;    
    
    let requestOptions = {
        host: domain,
        path: endpoint,
        port: 443,
        method: "GET",
    };
    
    requestOptions.headers = headers;

    return new Promise((resolve, reject) => {
        https.request(requestOptions, response => {
            let data = "";
            response.on("data", chunk => {
                data += chunk;
            });
            response.on("end", () => {
                //const dataObject = JSON.parse(data);
                response.data = external?data:JSON.parse(data);
                resolve(response);
            });
        })
            .on("error", error => {
                reject(error);
            })
            .end();
    });
};

exports.handler = (event, context, callback) => {

    const corsHeaders = {
        "Access-Control-Allow-Origin": accessControlAllowOriginValue,
        "Access-Control-Allow-Headers": accessControlAllowHeadersValue
    };

    const repeatResponse = (response) => {
        let multiValueHeaders = {};

        for (const headerName in response.headers) {
            if (Array.isArray(response.headers[headerName])) {
                multiValueHeaders[headerName] = response.headers[headerName];
                delete response.headers[headerName];
            }
        }

        callback(null, {
            statusCode: response.statusCode,
            body: JSON.stringify(response.data),
            headers: { ...response.headers, ...corsHeaders },
            multiValueHeaders: multiValueHeaders,
        });
    };

    const sendError = (error) => {
        callback(null, {
            statusCode: "400",
            body: JSON.stringify(error),
            headers: corsHeaders,
        });
    };

    request(event.queryStringParameters, event.headers, event.body)
        .then((response) => {
            repeatResponse(response);
        })
        .catch(error => {
            sendError(error);
        });
};
```
## Configuration

You need to deploy/upload this custom element to any web server that support SSL, or you can take advantage of Netlify service:
[![Deploy to Netlify](https://www.netlify.com/img/deploy/button.svg)](https://app.netlify.com/start/deploy?repository=https://github.com/hzik/LinkChecker)
### Link checking

Since all custom elements need to run on https but not all links starts with this protocol or don't even support this protocol, you can't make requests directly from this element due to browser security protection.
Please create a simple request service that takes the url parameter from the POST request and makes another call to this url and sends back the return code of such request.

A .Net Core lambda and php sample of such a requester:

```C#
using Amazon.Lambda.Core;
using System.Net;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.Json.JsonSerializer))]


namespace urlchecker
{
    public class Function
    {
        public string FunctionHandler(FunctionInput input, ILambdaContext context)
        {
            int statusCode = (int)default(HttpStatusCode);
            string url = input.url;
            var request = HttpWebRequest.Create(url);
            request.Method = "HEAD";
            request.Timeout = 10000;

            try
            {
                using (var response = request.GetResponse() as HttpWebResponse)
                {
                    if (response != null)
                    {
                        statusCode = (int)response.StatusCode;
                        response.Close();
                    }
                }
            }

            catch (WebException e)
            {
                if (e.Status == WebExceptionStatus.ProtocolError)
                {
                    var response = e.Response as HttpWebResponse;

                    if (response != null)
                    {
                        statusCode = (int)response.StatusCode;
                        response.Close();
                    }
                    else
                    {
                        return "Unknown error encountered.";
                    }
                }
                else
                {
                    return "Unknown error encountered.";
                }
            }
            return statusCode.ToString();
        }
    }

    public class FunctionInput
    {
        public string url { get; set; }
    }
}
```

```php
<?php
header('Access-Control-Allow-Origin: *', false);
$post = explode("=", file_get_contents('php://input'));
if(!empty($post[1])) {
	$req = curl_init();
	curl_setopt_array($req, [
		CURLOPT_URL            => urldecode($post[1]),
		CURLOPT_CUSTOMREQUEST  => "GET",
		CURLOPT_RETURNTRANSFER => true,
		CURLOPT_CONNECTTIMEOUT => 10,
		CURLOPT_TIMEOUT => 10,
	]);
		
	curl_exec($req);
	$info = curl_getinfo($req);
	
	print_r($info['http_code']);
}
?>
```
