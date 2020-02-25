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

You can test it by configuring https://kentico.github.io/kontent-custom-element-samples/UniqueTextbox/unique_text.html url for your custom element.

### Link checking

Since all custom elements need to run on https but not all links starts with this protocol or don't even support this protocol, you can't make requests directly from this element due to browser security protection.
Please create a simple request service that takes the url parameter from the POST request and makes another call to this url and sends back the return code of such request.
A php sample of such a requester:

```php
<?php
header('Access-Control-Allow-Origin: https://yourcustomelementdomain.com/', false);
if(isset($_POST['url'])) {
	$req = curl_init();
	curl_setopt_array($req, [
		CURLOPT_URL            => $_POST['url'],
		CURLOPT_CUSTOMREQUEST  => "GET",
		CURLOPT_RETURNTRANSFER => true,
	]);
		
	curl_exec($req);
	$info = curl_getinfo($req);
	
	print_r($info['http_code']);
}
?>
```
