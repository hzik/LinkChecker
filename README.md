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
  - `HOST`: `preview-delivery.kontent.ai`
  - `PATH`: `/<Project ID>/items`

[![Deploy to Netlify](https://www.netlify.com/img/deploy/button.svg)](https://app.netlify.com/start/deploy?repository=https://github.com/hzik/LinkChecker/)
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
$post = json_decode(file_get_contents('php://input'),true);
if($post['url']) {
	$req = curl_init();
	curl_setopt_array($req, [
		CURLOPT_URL            => $post['url'],
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
