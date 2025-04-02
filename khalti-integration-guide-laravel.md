# Integrating Khalti Payment Gateway in Flutter with Laravel Backend

This guide walks you through implementing Khalti payment gateway in an e-commerce application using Laravel for the backend and Flutter for the mobile application.

## What is Khalti?

Khalti is a popular digital wallet and payment gateway service in Nepal that allows businesses to accept digital payments through various methods. It provides robust APIs and SDKs to integrate payment functionality into applications.

## System Architecture

Our integration will have:
1. **Laravel Backend**: Handles payment initiation, transaction records, and payment verification
2. **Flutter Frontend**: Provides the user interface and interacts with the Khalti SDK

## Prerequisites

- Basic knowledge of Laravel and Flutter
- A Khalti merchant account (register at [merchant.khalti.com](https://merchant.khalti.com))
- Laravel development environment (PHP 8.x, Composer)
- Flutter development environment

## Part 1: Setting Up the Laravel Backend

### Step 1: Create a New Laravel Project

```bash
composer create-project laravel/laravel khalti-payment-app
cd khalti-payment-app
```

### Step 2: Set Up the Database

Configure your database connection in the `.env` file:

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=khalti_payment
DB_USERNAME=your_db_username
DB_PASSWORD=your_db_password
```

### Step 3: Add Khalti Configuration

Add Khalti related environment variables in your `.env` file:

```
KHALTI_SECRET_KEY=your_khalti_secret_key_here
KHALTI_PUBLIC_KEY=your_khalti_public_key_here
KHALTI_APP_URL=https://your-app-domain.com
KHALTI_API_URL=https://khalti.com/api/v2
```

Create a configuration file for Khalti at `config/khalti.php`:

```php
<?php

return [
    'secret_key' => env('KHALTI_SECRET_KEY'),
    'public_key' => env('KHALTI_PUBLIC_KEY'),
    'app_url' => env('KHALTI_APP_URL'),
    'api_url' => env('KHALTI_API_URL', 'https://khalti.com/api/v2'),
    'verification_url' => env('KHALTI_API_URL', 'https://khalti.com/api/v2') . '/payment/verify/',
    'epayment_url' => env('KHALTI_API_URL', 'https://khalti.com/api/v2') . '/epayment/initiate/',
];
```

### Step 4: Create Database Models

Create the necessary migrations for handling payments:

```bash
php artisan make:model Order -m
php artisan make:model Payment -m
```

Update the migration files:

1. Update the `create_orders_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained();
            $table->string('order_number')->unique();
            $table->decimal('amount', 10, 2);
            $table->string('status')->default('pending');
            $table->json('items')->nullable();
            $table->text('notes')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

2. Update the `create_payments_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('payments', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained();
            $table->foreignId('user_id')->constrained();
            $table->string('transaction_id')->unique();
            $table->string('pidx')->nullable();
            $table->decimal('amount', 10, 2);
            $table->string('payment_method');
            $table->string('payment_status')->default('pending');
            $table->json('payment_details')->nullable();
            $table->timestamp('paid_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('payments');
    }
};
```

Run the migrations:

```bash
php artisan migrate
```

### Step 5: Set Up Models

Update the Order model (`app/Models/Order.php`):

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Order extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'order_number',
        'amount',
        'status',
        'items',
        'notes',
    ];

    protected $casts = [
        'items' => 'array',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function payments(): HasMany
    {
        return $this->hasMany(Payment::class);
    }
}
```

Update the Payment model (`app/Models/Payment.php`):

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Payment extends Model
{
    use HasFactory;

    protected $fillable = [
        'order_id',
        'user_id',
        'transaction_id',
        'pidx',
        'amount',
        'payment_method',
        'payment_status',
        'payment_details',
        'paid_at',
    ];

    protected $casts = [
        'payment_details' => 'array',
        'paid_at' => 'datetime',
    ];

    public function order(): BelongsTo
    {
        return $this->belongsTo(Order::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

### Step 6: Create KhaltiService

Create a service to handle Khalti API interactions:

```bash
mkdir -p app/Services
touch app/Services/KhaltiService.php
```

Add the following code to `app/Services/KhaltiService.php`:

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\Payment;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class KhaltiService
{
    protected $apiUrl;
    protected $secretKey;
    protected $publicKey;
    protected $appUrl;

    public function __construct()
    {
        $this->apiUrl = config('khalti.api_url');
        $this->secretKey = config('khalti.secret_key');
        $this->publicKey = config('khalti.public_key');
        $this->appUrl = config('khalti.app_url');
    }

    /**
     * Initiate a payment with Khalti
     *
     * @param Order $order
     * @return array
     */
    public function initiatePayment(Order $order)
    {
        try {
            // Generate unique transaction ID
            $transactionId = 'TXN-' . time() . '-' . $order->id;

            // Create payment record
            $payment = Payment::create([
                'order_id' => $order->id,
                'user_id' => $order->user_id,
                'transaction_id' => $transactionId,
                'amount' => $order->amount,
                'payment_method' => 'Khalti',
                'payment_status' => 'pending',
            ]);

            // Prepare payload for Khalti
            $payload = [
                'return_url' => $this->appUrl . '/payment/success?transaction_id=' . $transactionId,
                'website_url' => $this->appUrl,
                'amount' => $order->amount * 100, // Convert to paisa (1 NPR = 100 paisa)
                'purchase_order_id' => $transactionId,
                'purchase_order_name' => 'Order #' . $order->order_number,
            ];

            // Make request to Khalti
            $response = Http::withHeaders([
                'Authorization' => 'Key ' . $this->secretKey,
                'Content-Type' => 'application/json',
            ])->post(config('khalti.epayment_url'), $payload);

            if ($response->successful() && isset($response['pidx'])) {
                // Update payment with pidx
                $payment->update([
                    'pidx' => $response['pidx'],
                    'payment_details' => $response->json(),
                ]);

                return [
                    'success' => true,
                    'data' => [
                        'pidx' => $response['pidx'],
                        'transaction_id' => $transactionId,
                        'payment_url' => $response['payment_url'] ?? null,
                    ],
                    'message' => 'Payment initiated successfully',
                ];
            }

            // Handle error
            Log::error('Khalti payment initiation failed', [
                'order' => $order->id,
                'response' => $response->json(),
            ]);

            return [
                'success' => false,
                'message' => 'Failed to initiate payment with Khalti',
                'errors' => $response->json(),
            ];
        } catch (\Exception $e) {
            Log::error('Exception during Khalti payment initiation', [
                'message' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]);

            return [
                'success' => false,
                'message' => 'An error occurred while initiating payment',
                'errors' => $e->getMessage(),
            ];
        }
    }

    /**
     * Verify a Khalti payment
     *
     * @param string $transactionId
     * @param string $status
     * @return array
     */
    public function verifyPayment(string $transactionId, string $status)
    {
        try {
            // Find the payment record
            $payment = Payment::where('transaction_id', $transactionId)->first();

            if (!$payment) {
                return [
                    'success' => false,
                    'message' => 'Payment record not found',
                ];
            }

            if ($status !== 'Completed') {
                return [
                    'success' => false,
                    'message' => 'Payment was not completed',
                ];
            }

            // Update payment status
            $payment->update([
                'payment_status' => 'paid',
                'paid_at' => now(),
            ]);

            // Update order status
            $payment->order->update([
                'status' => 'processing',
            ]);

            return [
                'success' => true,
                'message' => 'Payment verified successfully',
                'data' => [
                    'order_id' => $payment->order_id,
                    'transaction_id' => $payment->transaction_id,
                ]
            ];
        } catch (\Exception $e) {
            Log::error('Exception during Khalti payment verification', [
                'message' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]);

            return [
                'success' => false,
                'message' => 'An error occurred while verifying payment',
                'errors' => $e->getMessage(),
            ];
        }
    }
}
```

### Step 7: Create Controller

Create a controller to handle payment endpoints:

```bash
php artisan make:controller API/KhaltiController
```

Add the following code to `app/Http/Controllers/API/KhaltiController.php`:

```php
<?php

namespace App\Http\Controllers\API;

use App\Http\Controllers\Controller;
use App\Models\Order;
use App\Services\KhaltiService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class KhaltiController extends Controller
{
    protected $khaltiService;

    public function __construct(KhaltiService $khaltiService)
    {
        $this->khaltiService = $khaltiService;
    }

    /**
     * Initiate a payment with Khalti
     *
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function initiatePayment(Request $request)
    {
        // Validate request
        $validator = Validator::make($request->all(), [
            'order_id' => 'required|exists:orders,id',
            'payment_method' => 'required|in:Khalti',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors(),
            ], 422);
        }

        // Get the order
        $order = Order::findOrFail($request->order_id);

        // Initiate payment
        $result = $this->khaltiService->initiatePayment($order);

        if ($result['success']) {
            return response()->json($result, 200);
        }

        return response()->json($result, 500);
    }

    /**
     * Verify a payment
     *
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function verifyPayment(Request $request)
    {
        // Validate request
        $validator = Validator::make($request->all(), [
            'transaction_id' => 'required|exists:payments,transaction_id',
            'status' => 'required|string',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors(),
            ], 422);
        }

        // Verify payment
        $result = $this->khaltiService->verifyPayment(
            $request->transaction_id,
            $request->status
        );

        if ($result['success']) {
            return response()->json($result, 200);
        }

        return response()->json($result, 400);
    }

    /**
     * Handle payment success callback from Khalti
     *
     * @param Request $request
     * @return \Illuminate\Http\RedirectResponse
     */
    public function paymentSuccess(Request $request)
    {
        if ($request->has('transaction_id')) {
            $result = $this->khaltiService->verifyPayment(
                $request->transaction_id,
                'Completed'
            );

            if ($result['success']) {
                return redirect()->route('payment.success')->with('message', 'Payment completed successfully!');
            }
        }

        return redirect()->route('payment.failed')->with('message', 'Payment verification failed.');
    }
}
```

### Step 8: Define Routes

Add routes in `routes/api.php`:

```php
<?php

use App\Http\Controllers\API\KhaltiController;
use Illuminate\Support\Facades\Route;

Route::prefix('khalti')->group(function () {
    Route::post('/initiate-payment', [KhaltiController::class, 'initiatePayment']);
    Route::post('/verify-payment', [KhaltiController::class, 'verifyPayment']);
});
```

Add routes in `routes/web.php` for handling redirects:

```php
<?php

use App\Http\Controllers\API\KhaltiController;
use Illuminate\Support\Facades\Route;

Route::get('/payment/success', [KhaltiController::class, 'paymentSuccess'])->name('payment.success');
Route::get('/payment/success', function() {
    return view('payment.success');
})->name('payment.success');

Route::get('/payment/failed', function() {
    return view('payment.failed');
})->name('payment.failed');
```

### Step 9: Create Views (Optional)

Create simple success and failed views for payment redirects:

```bash
mkdir -p resources/views/payment
touch resources/views/payment/success.blade.php
touch resources/views/payment/failed.blade.php
```

Add content to `resources/views/payment/success.blade.php`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Payment Success</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            background-color: #f5f5f5;
        }
        .container {
            text-align: center;
            background-color: white;
            padding: 40px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
            max-width: 500px;
        }
        .success-icon {
            color: #4CAF50;
            font-size: 60px;
            margin-bottom: 20px;
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
            margin-bottom: 30px;
        }
        .btn {
            background-color: #56328c;
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 5px;
            text-decoration: none;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="success-icon">✓</div>
        <h1>Payment Successful</h1>
        <p>Your payment has been processed successfully. Thank you for your purchase!</p>
        <a href="/" class="btn">Return to Home</a>
    </div>
</body>
</html>
```

Add content to `resources/views/payment/failed.blade.php`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Payment Failed</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            background-color: #f5f5f5;
        }
        .container {
            text-align: center;
            background-color: white;
            padding: 40px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
            max-width: 500px;
        }
        .failed-icon {
            color: #f44336;
            font-size: 60px;
            margin-bottom: 20px;
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
            margin-bottom: 30px;
        }
        .btn {
            background-color: #56328c;
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 5px;
            text-decoration: none;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="failed-icon">✗</div>
        <h1>Payment Failed</h1>
        <p>Your payment could not be processed. Please try again or contact support.</p>
        <a href="/" class="btn">Return to Home</a>
    </div>
</body>
</html>
```

## Part 2: Setting Up the Flutter App

### Step 1: Create a Flutter Project

```bash
flutter create my_ecommerce_app
cd my_ecommerce_app
```

### Step 2: Add Dependencies

Update the `pubspec.yaml` file:

```yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
  dio: ^5.7.0
  provider: ^6.1.2
  khalti_checkout_flutter: ^1.0.0-dev.8
  shared_preferences: ^2.3.3
```

Run:
```bash
flutter pub get
```

### Step 3: Create API Service

Create a file `lib/services/api_service.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:shared_preferences/shared_preferences.dart';

class ApiService {
  final Dio _dio = Dio();
  final String _baseUrl = 'https://your-laravel-api.com/api';

  ApiService() {
    _dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: (options, handler) async {
          // Get the auth token from shared preferences
          final prefs = await SharedPreferences.getInstance();
          final token = prefs.getString('auth_token');
          
          if (token != null) {
            options.headers['Authorization'] = 'Bearer $token';
          }
          
          return handler.next(options);
        },
        onError: (DioException error, handler) {
          // Handle errors here
          return handler.next(error);
        },
      ),
    );
  }

  // Initiate Khalti payment
  Future<Map<String, dynamic>> initiateKhaltiPayment(int orderId) async {
    try {
      final response = await _dio.post(
        '$_baseUrl/khalti/initiate-payment',
        data: {
          'order_id': orderId,
          'payment_method': 'Khalti',
        },
      );
      
      return response.data;
    } catch (e) {
      if (e is DioException) {
        return {
          'success': false,
          'message': e.response?.data['message'] ?? 'An error occurred',
          'errors': e.response?.data['errors'],
        };
      }
      
      return {
        'success': false,
        'message': e.toString(),
      };
    }
  }

  // Verify Khalti payment
  Future<Map<String, dynamic>> verifyKhaltiPayment(
    String transactionId,
    String status,
  ) async {
    try {
      final response = await _dio.post(
        '$_baseUrl/khalti/verify-payment',
        data: {
          'transaction_id': transactionId,
          'status': status,
        },
      );
      
      return response.data;
    } catch (e) {
      if (e is DioException) {
        return {
          'success': false,
          'message': e.response?.data['message'] ?? 'An error occurred',
          'errors': e.response?.data['errors'],
        };
      }
      
      return {
        'success': false,
        'message': e.toString(),
      };
    }
  }
}
```

### Step 3: Create a Payment Provider

Create `lib/providers/payment_provider.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:dio/dio.dart';

class PaymentProvider extends ChangeNotifier {
  final Dio _dio = Dio();
  final String _baseUrl = 'http://your-server.com/api';
  
  String _pidx = '';
  String _transactionId = '';
  bool _isLoading = false;
  String _error = '';

  String get pidx => _pidx;
  String get transactionId => _transactionId;
  bool get isLoading => _isLoading;
  String get error => _error;

  // Initialize payment
  Future<bool> initiateKhaltiPayment(
      int userId, int productId, double amount) async {
    _isLoading = true;
    _error = '';
    notifyListeners();

    try {
      final response = await _dio.post(
        '$_baseUrl/khalti/initiate-payment',
        data: {
          'user_id': userId,
          'product_id': productId,
          'amount': amount,
          'payment_method': 'Khalti'
        },
      );

      if (response.statusCode == 200 && response.data['status'] == 'success') {
        _pidx = response.data['data']['pidx'];
        _transactionId = response.data['data']['transaction_id'];
        _isLoading = false;
        notifyListeners();
        return true;
      } else {
        _error = response.data['message'] ?? 'Failed to initiate payment';
        _isLoading = false;
        notifyListeners();
        return false;
      }
    } catch (e) {
      _error = e.toString();
      _isLoading = false;
      notifyListeners();
      return false;
    }
  }

  // Verify payment
  Future<bool> verifyPayment(String transactionId, String status) async {
    _isLoading = true;
    notifyListeners();

    try {
      final response = await _dio.post(
        '$_baseUrl/khalti/verify-payment',
        data: {
          'transaction_id': transactionId,
          'status': status,
        },
      );

      _isLoading = false;
      
      if (response.statusCode == 200 && response.data['status'] == 'success') {
        notifyListeners();
        return true;
      } else {
        _error = response.data['message'] ?? 'Payment verification failed';
        notifyListeners();
        return false;
      }
    } catch (e) {
      _error = e.toString();
      _isLoading = false;
      notifyListeners();
      return false;
    }
  }

  // Clear payment data
  void clearPaymentData() {
    _pidx = '';
    _transactionId = '';
    _error = '';
    notifyListeners();
  }
}
```

### Step 4: Create a Khalti Payment Screen

Create `lib/screens/khalti_payment_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:khalti_checkout_flutter/khalti_checkout_flutter.dart';
import 'package:provider/provider.dart';
import '../providers/payment_provider.dart';

class KhaltiPaymentScreen extends StatefulWidget {
  final int userId;
  final int productId;
  final double amount;
  final String productName;

  const KhaltiPaymentScreen({
    Key? key,
    required this.userId,
    required this.productId,
    required this.amount,
    required this.productName,
  }) : super(key: key);

  @override
  State<KhaltiPaymentScreen> createState() => _KhaltiPaymentScreenState();
}

class _KhaltiPaymentScreenState extends State<KhaltiPaymentScreen> {
  late Future<Khalti?> khaltiFuture;
  PaymentResult? paymentResult;

  @override
  void initState() {
    super.initState();
    
    // Initialize payment on screen load
    WidgetsBinding.instance.addPostFrameCallback((_) async {
      final provider = context.read<PaymentProvider>();
      
      // Initiate payment through the backend
      final success = await provider.initiateKhaltiPayment(
        widget.userId,
        widget.productId,
        widget.amount,
      );
      
      if (success) {
        // If payment initiation successful, initialize Khalti SDK
        setState(() {
          khaltiFuture = _initializeKhalti();
        });
      }
    });
  }

  // Initialize Khalti SDK
  Future<Khalti?> _initializeKhalti() async {
    final provider = Provider.of<PaymentProvider>(context, listen: false);
    final pidx = provider.pidx;
    final transactionId = provider.transactionId;
    
    if (pidx.isEmpty) {
      return null;
    }

    final payConfig = KhaltiPayConfig(
      publicKey: 'your_public_key_here', // Replace with your public key
      pidx: pidx,
      environment: Environment.test, // Use Environment.prod for production
    );

    final khalti = Khalti.init(
      enableDebugging: true,
      payConfig: payConfig,
      onPaymentResult: (paymentResult, khalti) async {
        setState(() {
          this.paymentResult = paymentResult;
        });
        
        // Verify payment with backend
        if (paymentResult.payload?.status == 'Completed') {
          await provider.verifyPayment(transactionId, "Completed");
          // Show success message
          if (!mounted) return;
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Payment Successful')),
          );
          
          // Navigate back or to success page
          if (!mounted) return;
          Navigator.pop(context, true);
        } else {
          // Show failure message
          if (!mounted) return;
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Payment Failed')),
          );
        }

        khalti.close(context);
      },
      onMessage: (khalti, {
        description,
        statusCode,
        event,
        needsPaymentConfirmation,
      }) async {
        print('Description: $description, Status Code: $statusCode');
        khalti.close(context);
      },
      onReturn: () => print('Successfully redirected to return_url.'),
    );

    return khalti;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Khalti Payment'),
      ),
      body: Consumer<PaymentProvider>(
        builder: (context, provider, child) {
          if (provider.isLoading) {
            return const Center(child: CircularProgressIndicator());
          }
          
          if (provider.error.isNotEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(
                    'Error: ${provider.error}',
                    style: const TextStyle(color: Colors.red),
                  ),
                  const SizedBox(height: 20),
                  ElevatedButton(
                    onPressed: () => Navigator.pop(context),
                    child: const Text('Go Back'),
                  ),
                ],
              ),
            );
          }

          return FutureBuilder<Khalti?>(
            future: khaltiFuture,
            builder: (context, snapshot) {
              if (snapshot.connectionState == ConnectionState.waiting) {
                return const Center(child: CircularProgressIndicator());
              }
              
              if (snapshot.hasError) {
                return Center(child: Text('Error: ${snapshot.error}'));
              }
              
              final khaltiInstance = snapshot.data;
              if (khaltiInstance == null) {
                return const Center(child: Text('Failed to initialize payment'));
              }

              return Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Image.asset(
                      'assets/images/logo.png', // Your app logo
                      height: 100,
                    ),
                    const SizedBox(height: 30),
                    Text(
                      widget.productName,
                      style: const TextStyle(
                        fontSize: 24,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    const SizedBox(height: 10),
                    Text(
                      'Rs. ${widget.amount}',
                      style: const TextStyle(fontSize: 22),
                    ),
                    const SizedBox(height: 30),
                    ElevatedButton(
                      style: ElevatedButton.styleFrom(
                        backgroundColor: const Color(0xFF56328c),
                        padding: const EdgeInsets.symmetric(
                          horizontal: 40,
                          vertical: 15,
                        ),
                      ),
                      onPressed: () => khaltiInstance.open(context),
                      child: const Text(
                        'Pay with Khalti',
                        style: TextStyle(fontSize: 18),
                      ),
                    ),
                    if (paymentResult != null) ...[
                      const SizedBox(height: 30),
                      Container(
                        padding: const EdgeInsets.all(15),
                        decoration: BoxDecoration(
                          border: Border.all(color: Colors.grey),
                          borderRadius: BorderRadius.circular(10),
                        ),
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text('Transaction ID: ${paymentResult!.payload?.transactionId ?? 'N/A'}'),
                            Text('Status: ${paymentResult!.payload?.status ?? 'N/A'}'),
                            Text('Amount: ${paymentResult!.payload?.totalAmount ?? 'N/A'}'),
                          ],
                        ),
                      ),
                    ],
                    const SizedBox(height: 20),
                    TextButton(
                      onPressed: () => Navigator.pop(context),
                      child: const Text('Cancel Payment'),
                    ),
                  ],
                ),
              );
            },
          );
        },
      ),
    );
  }
}
```

### Step 5: Update Main.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'screens/khalti_payment_screen.dart';
import 'providers/payment_provider.dart';

void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => PaymentProvider()),
        // Add other providers as needed
      ],
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Khalti Payment Demo',
      theme: ThemeData(
        primarySwatch: Colors.purple,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const ProductScreen(),
    );
  }
}

class ProductScreen extends StatelessWidget {
  const ProductScreen({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Product Details'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text(
              'Premium Membership',
              style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 20),
            const Text(
              'Rs. 1000',
              style: TextStyle(fontSize: 22),
            ),
            const SizedBox(height: 30),
            const Text(
              'Get access to all premium features',
              textAlign: TextAlign.center,
              style: TextStyle(fontSize: 16),
            ),
            const SizedBox(height: 50),
            ElevatedButton(
              style: ElevatedButton.styleFrom(
                backgroundColor: const Color(0xFF56328c),
                padding: const EdgeInsets.symmetric(
                  horizontal: 30,
                  vertical: 15,
                ),
              ),
              onPressed: () async {
                final result = await Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => const KhaltiPaymentScreen(
                      userId: 1, // Get from user session
                      productId: 1, // Get from product details
                      amount: 1000.0, // Get from product price
                      productName: 'Premium Membership',
                    ),
                  ),
                );
                
                if (result == true) {
                  // Payment successful, perform any post-payment actions
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(
                      content: Text('Membership activated successfully!'),
                      backgroundColor: Colors.green,
                    ),
                  );
                }
              },
              child: const Text(
                'Buy Now',
                style: TextStyle(fontSize: 18),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Part 3: Creating a Khalti Merchant Account

1. Sign up at [merchant.khalti.com](https://merchant.khalti.com/)
2. Complete verification to get your merchant account approved
3. Create a new merchant and get your API keys (test and live)
4. Configure webhook URLs (optional for advanced integration)

## Part 4: Testing the Integration

### Test Environment

1. Use the test environment by setting `environment: Environment.test` in the Flutter SDK
2. Use test credentials provided by Khalti (usually phone: 9800000000, MPIN: 1111, OTP: 987654)

### Testing Flow

1. Initiate a purchase from your Flutter app
2. The app will call your backend to create a transaction
3. The Khalti SDK will open a payment interface
4. Complete the payment with test credentials
5. Khalti will redirect back to your app
6. Your app will verify the payment with your backend
7. Your backend will update the order status

## Part 5: Common Issues and Troubleshooting

### Backend Issues

- **API Key Issues**: Ensure your Khalti secret key is correct
- **CORS Errors**: Configure proper CORS headers in your Express app
- **Database Connection**: Verify your database is properly connected

### Flutter Issues

- **SDK Compatibility**: Make sure you're using a compatible version of the Khalti SDK
- **State Management**: Use proper state management to handle payment states
- **Payment Callbacks**: Ensure all SDK callbacks are properly handled

## Part 6: Going to Production

1. Update environment to production:
   - Backend: Use the production Khalti API URL
   - Flutter: Set `environment: Environment.prod` in the SDK
2. Update API keys to live keys from your Khalti merchant dashboard
3. Implement robust error handling and logging
4. Add comprehensive payment analytics
5. Implement a retry mechanism for failed payments

## Conclusion

Integrating Khalti with Flutter and Node.js provides a seamless payment experience for your e-commerce application. This implementation can be adapted for various use cases beyond membership plans, such as product purchases, service bookings, or digital content sales.

By following this guide, you've set up a complete payment flow that allows your application to securely process payments, track transactions, and update order statuses automatically.

Remember to follow payment security best practices, keep your dependencies updated, and always test thoroughly before going to production.

---

Happy coding!
