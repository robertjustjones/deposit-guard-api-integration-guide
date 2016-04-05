![DepositGuard Logo"] ( https://www.depositguard.com/img/logo.gif "DepositGuard Logo" )

[TODO]: https://api-test.depositguard.net/images/g.png
[DOCS]: http://docs.depositguard.com

# Partner Integration Flow for the DepositGuard API

## Overview

The DepositGuard API offers simple and secure ways to manage funds for vacation and rental properties.

This guide is designed to be an overview of the implementation necessary for our Partner to integrate with the API. It is assumed that the Partner has an existing server-backed website that allows to add or modify certain client-side web forms and to add certain server-side http requests. DepositGuard is designed to be integrated simply into your existing website server flow.

The following discussion will describe:
- API Authentication
- Create Rental Agreement
- Setup Renter's Credit Card
- Request Deposit/Payment Charge


For detailed descriptions of the requests and responses, please refer to the [DepositGuard API Reference][DOCS].



## API Authentication

Before any other call can be made to the DepositGuard API, the Partner server must authenticate. The DepositGuard API uses the OAuth 2.0 "Client Credentials" protocol for server authentication. The flow requires that the password-based authentication be done from a protected server, and specifically not from a browser client.

### Server Authentication
When access to the API is needed, use the `POST /oauth/token` call as shown below with the `Content-Type: application/x-www-form-urlencoded` with the body `grant_type=client_credential`.  Use HTTP basic authorization with the Client Id and Secret supplied by DepositGuard.

```
POST /oauth/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <Base64 encoded string of “clientId:secret”>
Host: api.depositguard.net

grant_type=client_credentials


HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "<Access-Token>",
  "token_type": "Bearer",
  "expires_in": 28800
}
```

The `access_token` in the response body can be saved and reused until it expires, usually in 8 hours.  All other API request should be made with the authorization header: `Authorization: Bearer <Access-Token>`.


### Client Authentication
When the Partner server needs to render the Credit Card Setup page, the server must include a client access key.  Use the `POST /oauth/key` call as shown below. 

```
POST /oauth/key HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <Base64 encoded string of “clientId:secret”>
Host: api.depositguard.com

grant_type=client_credentials


HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_key": "<Access-Key>",
  "expires_in": 28800
}
```

The `access_token` in the response body can be saved and reused until it expires, usually in 8 hours. The value of the `access_token` should be rendered in the Credit Card Setup form as the `access_key` hidden input field as described in Step 2 below.  



## Step 1. Create Rental Agreement

When an End User has committed to renting a property for specific date range, use the `POST /agreements` call from the Partner server as shown below to record the necessary data. The `external_id` is optional and can be used to link to a user id on the Partner server.

```
POST /agreements HTTP/1.1
Content-Type: application/json
Authorization: Bearer <Access-Token>
Host: api.depositguard.com

{
  "external_id": "123456",
  "terms": {
    "dates":{
       "start_date": "YYYY-MM-dd 12:34:56Z",
       "end_date": "YYYY-MM-dd 12:34:56Z"
    },
    "text": "terms of the agreement go here"
  },
  "renter": [{
    "first_name": "foo",
    "last_name": "bar",
    "phone": "1234567890",
    "email": "renter@gmail.com",
    "address": {
      "city": "Austin",
      "state": "TX",
      "postal_code": "78704",
      "address_line1": "123 4th Street",
      "address_line2": "Apt. 456"
    }
  }],
  "landlord":[{
    "first_name": "foo",
    "last_name": "bar",
    "phone": "1234567890",
    "email": "landlord@gmail.com",
    "address": {
      "city": "Austin",
      "state": "TX",
      "postal_code": "78704",
      "address_line1": "123 4th Street",
      "address_line2": "Apt. 456"
    }
  }]
}


HTTP/1.1 201 Created
Location:  https://api.depositguard.com/agreements/d646e985-b895-4fb6-b121-b221e5daa9db

```

The `agreement_id` for the newly created agreement will be returned in the `Location` header. As shown in the example, the `agreement_id` is `d646e985-b895-4fb6-b121-b221e5daa9db`. Save the `agreement_id` on the Partner server for use in Step 3.

After the DepositGuard agreement has been created, the Partner will probably direct the Renter to a Credit Card setup page.

## Step 2. Setup Renter's Credit Card

To set up a renter's credit card requires both a client-side web form and server-side nonce processing.

### A. Client-side web form

When an End User is ready to setup a credit card for a payment to be processed by DepositGuard, the Partner should render a client-side web form modeled on the following template. 

Other than a few key elements that should not change, the Partner is free to design the form to match existing style.  The `action=` and `method=` attributes of the `<form>` element must remain as shown. The `name` attribute for each of the `<input>` element must match the names shown. Both `"hidden"` fields are required. The value for the `"access_key"` field must be an unexpired Client Access Key as returned from the `POST /ouath/key` call in "API Authentication". The value of the `"redirect_url"` field must point to the handler described in "Server-side nonce processing". Any additional fields will be ignored. The `<input type="submit" ...>` element is required unless javascript is used to mimic the POST behavior.  

```
<html>

<form action="http://api.depositguard.net/paymentmethods" 
  method="POST"> 
  <div>
    <label for="name">Name:</label>
    <input type="text" name="name"/>
  </div>
  <div>
    <label for="cc_number">cc_number:</label>
    <input type="text" name="cc_number"/>
  </div>
  <div>
    <label for="cvv">cvv:</label>
    <input type="text" name="cvv"/>
  </div>
  <div>
    <label for="city">city:</label>
    <input type="text" name="city"/>
  </div>
  <div>
    <label for="state">state:</label>
    <input type="text" name="state"/>
  </div>
  <div>
    <label for="postal_code">postal_code:</label>
    <input type="text" name="postal_code"/>
  </div>
  <div>
    <label for="address_line_1">address_line_1:</label>
    <input type="text" name="address_line_1"/>
  </div>
  <div>
    <label for="address_line_2">address_line_2:</label>
    <input type="text" name="address_line_2"/>
  </div>
  <div>
    <input type="hidden" name="access_key" value="ABC_XYX_123"/>
    <input type="hidden" name="redirect_url" value="https://www.merchantsite.com/process_nonce"/>
    <input type="submit" value="Save Payment Method"/> 
  </div>
</form>

</html>
```

### B. Server-side nonce processing


When the end user's Credit Card setup form is submitted, DepositGuard confirms the credit card and returns a user-specific payment method nonce to the redirect url provided.  Posting to the rediect_url allows the Partner server to confirm the legitimacy of the User Session (cookies) before finally activating the payment method.  Activate the nonce with a server-side call and store it for re-use. Submit a payment request using the nonce to identify the payment method. The response will confirm if the payment was successful or respond with an appropriate error message.

```
POST /nonces/paymentmethods HTTP/1.1
Content-Type: application/json
Authorization: Bearer <Access-Token>
Host: api.depositguard.com


{
  "nonce": "<payment-method-nonce>"
}
```

Once the nonce is confirmed, the DepositGuard payment method setup is complete and ready for use. The Partner next will probably direct the Renter to a Payment page.


## Step 3. Request Deposit/Payment Charge

Once an agreement has been created and a payment method is activated, the Partner may present a payment page to the Renter in any suitable fashion, depending on the Partner's workflow. Submitting the request authorizes and immediately charges a payment for the specified amount. The value is held in escrow until a payout is disbursed.

When ready, the Partner makes a server-side `POST /agreements/{agreement_id}/deposits` request. The JSON body includes the `payment_method_nonce` received in Step 2-B. The `type` field can be `rental` or `security_deposit`.  The `note` field is used to further describe the payment.


```
POST /agreements/d646e985-b895-4fb6-b121-b221e5daa9db/deposits HTTP/1.1
Content-Type: application/json
Authorization: Bearer <Access-Token>
Host: api.depositguard.com

{
  "payment_method_nonce": "6dcbd862-e01d-4639-a5b4-774bda55ed09",
  "amount": {
    "total": "123.45",
    "currency": "USD"
  },
  "type": "security_deposit",
  "note": "half of deposit"
}


HTTP/1.1 201 Created
Location:  https://api.depositguard.com/agreements/d646e985-b895-4fb6-b121-b221e5daa9db/deposits/eaf557ca-ee95-4939-951a-bd2b2e1047ce
```

Remember that the renter will receive their security deposit back on the card(s) used at the time that agreement is completed.


## Appendix A. 
![Overview Sequence][overviewSequence]
[overviewSequence]: https://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgUGFydG5lciBJbnRlZ3JhdGlvbiBGbG93IGZvciB0aGUgRGVwb3NpdEd1YXJkIEFQSQoKcGFydGljaXBhbnQgRW5kLVVzZXIgQnJvd3NlcgAQDQBUCFNlcnYADA8ARBIKbm90ZSBvdmVyAEQRLAA7DiwAgQMQOiBBUEkgQXV0aGVudGljAIFABQBECwAdIQCBGgYAMRAAgTAOLT4AZBJQT1NUIC9vYXV0aC90b2tlbiAoSWQvU2VjcmV0KQoAgikQLT4AggIOOiAALQdnb29kAIJqBTggaHJzKQCCAwZyaWdodCBvZgCCNg8AgSQJAGoGdXNlADUGYWxsIHMAgUEGcmVxdWVzdHNcbiAqZXhjZXB0KgCBIwdwYXltZW50bWV0aG9kcy4AgXcsIENsaWVudCBBY2Nlc3MgS2V5AIFxL2tleQCBbyNhAFkGa2V5AIFnHwCBCAcAJgoAggUFb25seQCFMAliAIUMBlxuAIFpFCBzdWJtaXQuAIQ5PzEuIENyZWF0ZSBSZW50ZXIvTGFuZGxvcmQACwVhbCBBZ3JlZW1lbnQKAIYaEACDfRIAhGERAIZNEDogAIYKGzogRW5kIACHBwVzZWxlY3RzIHByb3BlcnR5XG4gKHRlcm1zLCByAIEiBSwgbACBIQcpAHUjAIViBXIAgUoGZm9ybQCHHw0AhnohAIIZB2EAggAJAIYwKAApCXMge2RhdGF9AIQcJACCZwhJZCAoTG8AiAUGKQCCNiNTdQCFVAUAiEg_Mi4AhAEHIENyZWRpdCBDYXJkIFNldHVwCgCDXiNSAIc0BgArEiBGAIJxBgCDfyIAKRJQYWdlAIpBHACELQsAhT4Fc1xuIGMAgTgGY2FyZCBpbmZvAIUqEwCJdhgAiFYOAItKCwCEBRUAWQ0gdmFsaWRhdGVkXG4gICAgIGFuZCBzdG9yZWQgc2VjdXJlbACIMBQAhhQSMzA0IFJFRElSRUNUIHRvICJyZWRpcmVjdF91cmwiIHdpdGggIm5vbmNlIiBwYXJhbQCGeiMANwwAPAYAPAUAOQcAjFUYOiBTdG9yZQAlBwB0BXVzZXIgZGF0YQCMOCkAgSUFAIkwEHsAgTsFAIVwJDIwMCBPSwCFVy0Ajk89My4AkCwIIFAAjGsGIENoYXJnZQCFWAgAhV8rADkQAIYDBQCKBSIAIhUAhVUuIACOJgcAhWgYAIoQFQAsCACKIAYAiTcyL3tpZH0vZACSbAYAgzYuMSBDUkVBVEVEAIM7LgoKCgo&s=earth 
