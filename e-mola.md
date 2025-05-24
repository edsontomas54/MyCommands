# MOVITEL USSD PUSH API Integration for Laravel

This document outlines the steps to integrate the MOVITEL USSD PUSH API into a Laravel application.

## Table of Contents
1. [Configuration Setup](#configuration-setup)
2. [Service Class Implementation](#service-class-implementation)
3. [API Controller](#api-controller)
4. [Routing](#routing)
5. [Callback Processing](#callback-processing)
6. [Error Handling](#error-handling)
7. [Usage Examples](#usage-examples)
8. [Testing Considerations](#testing-considerations)
9. [Additional Recommendations](#additional-recommendations)

## Configuration Setup

Create a configuration file and add environment variables:

```php
// config/movitel.php
return [
    'wsdl_url' => env('MOVITEL_WSDL_URL'),
    'username' => env('MOVITEL_USERNAME'),
    'password' => env('MOVITEL_PASSWORD'),
    'partner_code' => env('MOVITEL_PARTNER_CODE'),
    'key' => env('MOVITEL_KEY'),
    'callback_url' => env('MOVITEL_CALLBACK_URL'),
];


Implementing MOVITEL USSD PUSH API in Laravel
Based on the specification document, I'll guide you through implementing the MOVITEL USSD PUSH API in your Laravel project. This implementation will cover the main functionalities including sending USSD push messages, checking transaction status, disbursements, and callbacks.

1. Setup Configuration
First, let's create a configuration file for the API credentials and endpoints.

php
// config/movitel.php
return [
    'wsdl_url' => env('MOVITEL_WSDL_URL', 'https://xxx.xxx.xxx.xxx:yyyy/BCCSGateway/BCCSGateway?wsdl'),
    'username' => env('MOVITEL_USERNAME'),
    'password' => env('MOVITEL_PASSWORD'),
    'partner_code' => env('MOVITEL_PARTNER_CODE'),
    'key' => env('MOVITEL_KEY'),
    'callback_url' => env('MOVITEL_CALLBACK_URL'),
];
Add these to your .env file:

MOVITEL_WSDL_URL=https://xxx.xxx.xxx.xxx:yyyy/BCCSGateway/BCCSGateway?wsdl
MOVITEL_USERNAME=your_username
MOVITEL_PASSWORD=your_password
MOVITEL_PARTNER_CODE=your_partner_code
MOVITEL_KEY=your_private_key
MOVITEL_CALLBACK_URL=https://yourdomain.com/api/movitel/callback

=============================>>>>>>>>>>>><<<<<<<<<<<<<<<<<<<<<<>>>>>>>>>>>>>>>>>>>>>


2. Create a Service Class
Create a service class to handle all MOVITEL API interactions.

php
// app/Services/MovitelService.php

namespace App\Services;

use SoapClient;
use Exception;

class MovitelService
{
    protected $client;
    protected $config;

    public function __construct()
    {
        $this->config = config('movitel');
        
        try {
            $this->client = new SoapClient($this->config['wsdl_url'], [
                'trace' => 1,
                'exceptions' => true,
            ]);
        } catch (Exception $e) {
            throw new Exception("Failed to initialize SOAP client: " . $e->getMessage());
        }
    }

    /**
     * Send USSD Push Message
     */
    public function sendUssdPush($msisdn, $message, $amount, $transId, $refNo, $language = 'en')
    {
        $params = [
            'partnerCode' => $this->config['partner_code'],
            'msisdn' => $msisdn,
            'smsContent' => $message,
            'transAmount' => $amount,
            'transId' => $transId,
            'language' => $language,
            'refNo' => $refNo,
            'key' => $this->config['key'],
        ];

        return $this->makeRequest('pushUsedMessage', $params);
    }

    /**
     * Check Transaction Status
     */
    public function checkTransactionStatus($transId)
    {
        $params = [
            'partnerCode' => $this->config['partner_code'],
            'transId' => $transId,
            'transType' => 'QUERY_TXN',
            'key' => $this->config['key'],
        ];

        return $this->makeRequest('pushUsedQueryTrans', $params);
    }

    /**
     * Make B2C Disbursement
     */
    public function makeDisbursement($msisdn, $amount, $transId, $remark = '')
    {
        $params = [
            'partnerCode' => $this->config['partner_code'],
            'msisdn' => $msisdn,
            'smsContent' => $remark ?: config('app.name'),
            'transAmount' => $amount,
            'transId' => $transId,
            'key' => $this->config['key'],
        ];

        return $this->makeRequest('pushUsedDisbursementB2C', $params);
    }

    /**
     * Query Beneficiary Name
     */
    public function queryBeneficiaryName($msisdn, $transId)
    {
        $params = [
            'transId' => $transId,
            'partnerCode' => $this->config['partner_code'],
            'msisdn' => $msisdn,
            'key' => $this->config['key'],
        ];

        return $this->makeRequest('queryBeneficiaryName', $params);
    }

    /**
     * Query Account Balance
     */
    public function queryAccountBalance($transId)
    {
        $params = [
            'partnerCode' => $this->config['partner_code'],
            'transId' => $transId,
            'key' => $this->config['key'],
        ];

        return $this->makeRequest('queryAccountBalance', $params);
    }

    /**
     * Generic SOAP Request Handler
     */
    protected function makeRequest($wsCode, array $params)
    {
        try {
            // Prepare the SOAP request
            $soapParams = [
                'username' => $this->config['username'],
                'password' => $this->config['password'],
                'wscode' => $wsCode,
            ];

            // Add params as <param> elements
            foreach ($params as $name => $value) {
                $soapParams['param'][] = [
                    'name' => $name,
                    'value' => $value,
                ];
            }

            // Make the SOAP call
            $response = $this->client->gwOperation(['input' => $soapParams]);

            // Parse the response
            return $this->parseResponse($response);

        } catch (Exception $e) {
            return [
                'error' => '9001', // Exception code from the spec
                'description' => $e->getMessage(),
                'success' => false,
            ];
        }
    }

    /**
     * Parse SOAP Response
     */
    protected function parseResponse($response)
    {
        if (!isset($response->Result)) {
            return [
                'error' => '4000', // Invalid format
                'description' => 'Invalid response format from MOVITEL API',
                'success' => false,
            ];
        }

        $result = $response->Result;

        // Check for gateway-level errors
        if ($result->error != '0') {
            return [
                'error' => $result->error,
                'description' => $result->description ?? 'Unknown error',
                'success' => false,
                'gwtransid' => $result->gwtransid ?? null,
            ];
        }

        // Parse the original XML response
        try {
            $original = simplexml_load_string($result->original);
            $return = $original->xpath('//return')[0];

            return [
                'error' => (string)$return->errorCode,
                'message' => (string)$return->message,
                'requestId' => (string)$return->requestId ?? null,
                'balance' => (string)($return->balance ?? null),
                'orgResponseCode' => (string)($return->orgResponseCode ?? null),
                'orgResponseMessage' => (string)($return->orgResponseMessage ?? null),
                'gwtransid' => (string)$result->gwtransid ?? null,
                'success' => ((string)$return->errorCode) === '0',
            ];

        } catch (Exception $e) {
            return [
                'error' => '6005', // Error when load response to DOM
                'description' => 'Failed to parse response: ' . $e->getMessage(),
                'success' => false,
                'gwtransid' => $result->gwtransid ?? null,
            ];
        }
    }
}
3. Create a Controller for API Endpoints
php
// app/Http/Controllers/Api/MovitelController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Services\MovitelService;
use Illuminate\Http\Request;

class MovitelController extends Controller
{
    protected $movitel;

    public function __construct(MovitelService $movitel)
    {
        $this->movitel = $movitel;
    }

    /**
     * Send USSD Push Message
     */
    public function sendUssdPush(Request $request)
    {
        $request->validate([
            'msisdn' => 'required|string|size:9',
            'message' => 'required|string|max:180',
            'amount' => 'required|numeric',
            'trans_id' => 'required|string|max:30',
            'ref_no' => 'required|string|max:20',
            'language' => 'sometimes|string|in:en,pt',
        ]);

        $response = $this->movitel->sendUssdPush(
            $request->msisdn,
            $request->message,
            $request->amount,
            $request->trans_id,
            $request->ref_no,
            $request->language ?? 'en'
        );

        return response()->json($response);
    }

    /**
     * Check Transaction Status
     */
    public function checkTransactionStatus(Request $request)
    {
        $request->validate([
            'trans_id' => 'required|string|max:30',
        ]);

        $response = $this->movitel->checkTransactionStatus($request->trans_id);

        return response()->json($response);
    }

    /**
     * Make Disbursement
     */
    public function makeDisbursement(Request $request)
    {
        $request->validate([
            'msisdn' => 'required|string|size:9',
            'amount' => 'required|numeric',
            'trans_id' => 'required|string|max:30',
            'remark' => 'sometimes|string|max:100',
        ]);

        $response = $this->movitel->makeDisbursement(
            $request->msisdn,
            $request->amount,
            $request->trans_id,
            $request->remark
        );

        return response()->json($response);
    }

    /**
     * Query Beneficiary Name
     */
    public function queryBeneficiaryName(Request $request)
    {
        $request->validate([
            'msisdn' => 'required|string|size:9',
            'trans_id' => 'required|string|max:30',
        ]);

        $response = $this->movitel->queryBeneficiaryName(
            $request->msisdn,
            $request->trans_id
        );

        return response()->json($response);
    }

    /**
     * Query Account Balance
     */
    public function queryAccountBalance(Request $request)
    {
        $request->validate([
            'trans_id' => 'required|string|max:30',
        ]);

        $response = $this->movitel->queryAccountBalance($request->trans_id);

        return response()->json($response);
    }

    /**
     * Handle Callback from MOVITEL
     */
    public function handleCallback(Request $request)
    {
        $request->validate([
            'requestId' => 'required|string',
            'transId' => 'required|string',
            'refNo' => 'required|string',
            'errorCode' => 'required|string',
            'message' => 'required|string',
        ]);

        // Process the callback (save to database, trigger events, etc.)
        // Example:
        \Log::info('MOVITEL Callback Received:', $request->all());
        
        // You might want to dispatch a job to handle this async
        // \App\Jobs\ProcessMovitelCallback::dispatch($request->all());

        return response()->json([
            'ResponseCode' => '0',
            'ResponseMessage' => 'Callback received successfully',
        ]);
    }
}
4. Add API Routes
php
// routes/api.php

use App\Http\Controllers\Api\MovitelController;

Route::prefix('movitel')->group(function () {
    Route::post('/push', [MovitelController::class, 'sendUssdPush']);
    Route::post('/check-status', [MovitelController::class, 'checkTransactionStatus']);
    Route::post('/disburse', [MovitelController::class, 'makeDisbursement']);
    Route::post('/query-beneficiary', [MovitelController::class, 'queryBeneficiaryName']);
    Route::post('/query-balance', [MovitelController::class, 'queryAccountBalance']);
    Route::post('/callback', [MovitelController::class, 'handleCallback']);
});
5. Create a Job for Async Callback Processing (Optional)
php
// app/Jobs/ProcessMovitelCallback.php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessMovitelCallback implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $callbackData;

    public function __construct(array $callbackData)
    {
        $this->callbackData = $callbackData;
    }

    public function handle()
    {
        // Process the callback data
        // Example: Update your transaction status in database
        \Log::info('Processing MOVITEL callback:', $this->callbackData);
        
        // You might want to:
        // 1. Find the transaction by transId
        // 2. Update its status based on errorCode
        // 3. Notify users if needed
    }
}
6. Error Handling and Response Codes
Create a helper class to handle MOVITEL error codes:

php
// app/Helpers/MovitelHelper.php

namespace App\Helpers;

class MovitelHelper
{
    public static function getErrorMessage($errorCode)
    {
        $errors = [
            '0' => 'Success when calling detail api',
            '2007' => 'Time out when calling detail api.',
            '1000' => 'Username is invalid',
            '2000' => 'Webservice code is invalid',
            '4000' => 'The request\'s message\'s format is not accurate',
            // Add all other error codes from the specification
            // ...
            
            // Detail response codes
            '98' => 'Login failed invalid user',
            '02' => 'Login failed.IP client not valid',
            '03' => 'Transaction has failed',
            '05' => 'Invalid partner code',
            '06' => 'Invalid amount',
            '07' => 'Invalid MSISDN',
            '08' => 'Text must be greater than 0 or equal to 150 characters',
            '09' => 'The transaction ID must be less than or equal to 50 characters',
            '10' => 'ISDN not in white list',
            '11' => 'Customer did not enter PIN',
            '12' => 'Customer dose not have eMola account',
            '13' => 'Cannot find org request id',
            '14' => 'TransID already existed',
            '15' => 'Request does not exist',
            '20' => 'The ISDN is in other process.',
            '22' => 'Push message done(When asynchronous is enabled)',
            '23' => 'Invalid refNo',
            '24' => 'The function is not supported',
            '25' => 'Invalid LOTO service',
            '26' => 'Invalid LOTO number',
            '27' => 'Payment request not found',
            '28' => 'Transaction has processed',
            '99' => 'Have an occur while processing',
        ];

        return $errors[$errorCode] ?? 'Unknown error';
    }
}
7. Usage Examples
Here's how you would use the service in your application:

php
// Example 1: Sending a USSD Push
$movitel = app(App\Services\MovitelService::class);
$response = $movitel->sendUssdPush(
    '861234567', // msisdn (must start with 86 or 87)
    'Please confirm your payment of 100 MZN', // message
    '100', // amount
    'TX123456789', // transaction ID
    'REF12345' // reference number
);

if ($response['success']) {
    // Transaction initiated successfully
    $requestId = $response['requestId'];
} else {
    $errorMessage = App\Helpers\MovitelHelper::getErrorMessage($response['error']);
}

// Example 2: Checking transaction status
$statusResponse = $movitel->checkTransactionStatus('TX123456789');

// Example 3: Handling callback (automatically handled by the controller)
8. Testing Considerations
Mock the SOAP Client: For testing, you should mock the SOAP client to avoid making real API calls.

Test Error Scenarios: Test various error responses from the API.

Validate Inputs: Ensure all inputs are properly validated according to the specification.

Logging: Implement detailed logging for debugging purposes.

9. Additional Recommendations
Database Transactions: Consider storing all requests and responses in your database for auditing.

Retry Mechanism: Implement retry logic for failed requests.

Queue Processing: Use Laravel queues for processing callbacks and other async operations.

Security: Ensure your callback endpoint is secured (IP whitelisting if possible).

Monitoring: Set up monitoring for failed transactions and alerts.

This implementation provides a solid foundation for integrating with the MOVITEL USSD PUSH API in your Laravel application. You may need to adjust it based on your specific requirements and the actual behavior of the MOVITEL API.




