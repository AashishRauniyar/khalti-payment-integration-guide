# Integrating Khalti Payment Gateway in Flutter and Node.js

In today's e-commerce landscape, providing seamless payment options is crucial for customer satisfaction. This guide will walk you through implementing Khalti payment gateway integration with a Flutter app and Node.js backend.

## What is Khalti?

Khalti is a popular digital wallet service and payment gateway in Nepal that allows businesses to accept digital payments. It provides robust APIs and mobile SDKs for integrating payment functionality into applications.

## System Overview

Here's what we'll build:
1. **Backend (Node.js/Express)**: Handles payment initiation, stores transaction details, and verifies payments
2. **Frontend (Flutter)**: Displays payment UI and processes payments through the Khalti SDK

## Prerequisites

- Basic knowledge of Flutter and Node.js
- A Khalti merchant account (register at [merchant.khalti.com](https://merchant.khalti.com))
- Flutter development environment
- Node.js development environment

## Part 1: Setting Up the Backend (Node.js)

### Step 1: Project Setup

Create a new Node.js project and install required dependencies:

```bash
mkdir khalti-payment-server
cd khalti-payment-server
npm init -y
npm install express prisma dotenv axios
```

### Step 2: Configure Environment Variables

Create a `.env` file in your project root:

```
DATABASE_URL="postgresql://username:password@localhost:5432/your_db_name"
NODE_ENV=development
KHALTI_SECRET_KEY="your_khalti_secret_key_here"
BASE_URL="http://your-server.com/"
```

### Step 3: Set Up Prisma (Database ORM)

Initialize Prisma:

```bash
npx prisma init
```

Create your database schema in `prisma/schema.prisma`. Here's a simplified version based on the provided files:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model memberships {
  membership_id Int               @id @default(autoincrement())
  user_id       Int?
  plan_id       Int
  start_date    DateTime?         @db.Date
  end_date      DateTime?         @db.Date
  status        membership_status @default(Pending)
  created_at    DateTime?         @default(now()) @db.Timestamp(6)
  updated_at    DateTime?         @default(now()) @db.Timestamp(6)

  payments        payments[]
  membership_plan membership_plan @relation(fields: [plan_id], references: [plan_id], onDelete: Cascade)
  users           users?          @relation(fields: [user_id], references: [user_id], onDelete: Cascade)
}

model payments {
  payment_id     Int            @id @default(autoincrement())
  membership_id  Int?
  user_id        Int?
  price          Decimal        @db.Decimal(10, 2)
  payment_method payment_method
  transaction_id String         @unique @db.VarChar(100)
  pidx           String?        @db.VarChar(100)
  payment_date   DateTime       @db.Date
  payment_status payment_status
  created_at     DateTime?      @default(now()) @db.Timestamp(6)
  memberships    memberships?   @relation(fields: [membership_id], references: [membership_id], onDelete: Cascade)
  users          users?         @relation(fields: [user_id], references: [user_id], onDelete: Cascade)
}

model users {
  user_id     Int           @id @default(autoincrement())
  full_name   String?       @db.VarChar(100)
  email       String        @unique @db.VarChar(255)
  password    String        @db.VarChar(255)
  memberships memberships[]
  payments    payments[]
}

enum membership_status {
  Pending
  Active
  Expired
  Cancelled
}

enum payment_method {
  Khalti
  Cash
}

enum payment_status {
  Paid
  Pending
  Failed
}
```

Generate Prisma client:

```bash
npx prisma generate
npx prisma db push
```

### Step 4: Create Khalti Controller

Create a file `controllers/khaltiController.js`:

```javascript
import { PrismaClient } from '@prisma/client';
import axios from 'axios';
import dotenv from 'dotenv';

dotenv.config();

const prisma = new PrismaClient();
const KHALTI_API_URL = process.env.NODE_ENV === "production"
    ? "https://khalti.com/api/v2/epayment/initiate/"
    : "https://dev.khalti.com/api/v2/epayment/initiate/";
const KHALTI_SECRET_KEY = process.env.KHALTI_SECRET_KEY;
const BASE_URL = process.env.BASE_URL;

export const initiatePayment = async (req, res) => {
    try {
        const { user_id, product_id, amount, payment_method } = req.body;

        if (payment_method !== 'Khalti') {
            return res.status(400).json({
                status: 'failure',
                message: 'Invalid payment method. Please use "Khalti".'
            });
        }

        // Validate user
        const user = await prisma.users.findUnique({ 
            where: { user_id: parseInt(user_id) } 
        });
        
        if (!user) {
            return res.status(404).json({ 
                status: 'failure', 
                message: 'User not found' 
            });
        }

        // Validate product (in this example, it's a membership plan)
        const product = await prisma.membership_plan.findUnique({ 
            where: { plan_id: parseInt(product_id) } 
        });
        
        if (!product) {
            return res.status(404).json({ 
                status: 'failure', 
                message: 'Product not found' 
            });
        }

        // Create a pending order/membership
        const newOrder = await prisma.memberships.create({
            data: {
                user_id: parseInt(user_id),
                plan_id: parseInt(product_id),
                status: 'Pending',
            }
        });

        // Generate unique transaction ID
        const transactionId = `TXN-${Date.now()}-${user_id}`;

        // Prepare Khalti payload
        const paymentPayload = {
            return_url: `${BASE_URL}payment-success?transaction_id=${transactionId}`,
            website_url: BASE_URL,
            amount: Math.round(amount * 100), // Convert to paisa (1 NPR = 100 paisa)
            purchase_order_id: transactionId,
            purchase_order_name: product.plan_type,
        };

        // Initiate payment with Khalti
        const khaltiResponse = await axios.post(KHALTI_API_URL, paymentPayload, {
            headers: {
                Authorization: `Key ${KHALTI_SECRET_KEY}`,
                "Content-Type": "application/json",
            }
        });

        if (!khaltiResponse.data.pidx) {
            throw new Error("Failed to get Khalti pidx");
        }

        // Create payment record
        await prisma.payments.create({
            data: {
                user_id: parseInt(user_id),
                membership_id: newOrder.membership_id,
                price: amount,
                payment_method: 'Khalti',
                transaction_id: transactionId,
                pidx: khaltiResponse.data.pidx,
                payment_status: 'Pending',
                payment_date: new Date()
            }
        });

        // Return response to client
        res.status(200).json({
            status: 'success',
            data: {
                pidx: khaltiResponse.data.pidx,
                transaction_id: transactionId,
                payment_url: khaltiResponse.data.payment_url
            },
            message: 'Payment initiated successfully',
        });

    } catch (error) {
        console.error("Error initiating payment:", error);
        res.status(500).json({
            status: 'failure',
            message: 'Internal server error',
            error: error.message
        });
    }
};

export const verifyPayment = async (req, res) => {
    try {
        const { transaction_id, status } = req.body;

        // Find payment record
        const payment = await prisma.payments.findUnique({
            where: { transaction_id },
            include: { memberships: true }
        });

        if (!payment) {
            return res.status(404).json({ 
                status: 'failure', 
                message: 'Payment not found' 
            });
        }

        // Check payment status
        if (status !== 'Completed') {
            return res.status(400).json({
                status: 'failure',
                message: 'Payment was not successful'
            });
        }

        // Update payment status
        await prisma.payments.update({
            where: { transaction_id },
            data: { payment_status: 'Paid' }
        });

        // Update order/membership status
        await prisma.memberships.update({
            where: { membership_id: payment.membership_id },
            data: {
                status: 'Active',
                start_date: new Date(),
                end_date: new Date(new Date().setMonth(new Date().getMonth() + 1))
            }
        });

        res.status(200).json({
            status: 'success',
            message: 'Payment verified, order activated'
        });

    } catch (error) {
        console.error("Error verifying payment:", error);
        res.status(500).json({
            status: 'failure',
            message: 'Internal server error',
            error: error.message
        });
    }
};
```

### Step 5: Set Up Routes

Create `routes/khaltiRoutes.js`:

```javascript
import express from 'express';
import { initiatePayment, verifyPayment } from '../controllers/khaltiController.js';

const khaltiRouter = express.Router();

khaltiRouter.post('/initiate-payment', initiatePayment);
khaltiRouter.post('/verify-payment', verifyPayment);

export default khaltiRouter;
```

### Step 6: Create Server Entry Point

Create `server.js`:

```javascript
import express from 'express';
import dotenv from 'dotenv';
import khaltiRouter from './routes/khaltiRoutes.js';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 5000;

// Middleware
app.use(express.json());

// Routes
app.use('/api/khalti', khaltiRouter);

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Part 2: Setting Up the Flutter App

### Step 1: Set Up a New Flutter Project

```bash
flutter create my_ecommerce_app
cd my_ecommerce_app
```

### Step 2: Add Dependencies

Update `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
  provider: ^6.1.2
  dio: ^5.7.0
  khalti_checkout_flutter: ^1.0.0-dev.8  # Khalti SDK
  go_router: ^14.6.1
  http: ^1.3.0
```

Run:
```bash
flutter pub get
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
