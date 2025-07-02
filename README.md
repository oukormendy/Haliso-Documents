# Kodo X - Backend and API Integration Plan

## Executive Summary

This document outlines the comprehensive backend architecture and API integration strategy for Kodo X, a mobile fintech platform supporting GMD and USD wallets, virtual cards, peer-to-peer transfers, and other financial services. The backend is built on Supabase, providing a scalable, secure foundation for all platform operations, with strategic integrations to third-party financial services.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Database Schema](#2-database-schema)
3. [Authentication & User Management](#3-authentication--user-management)
4. [Core Financial Services](#4-core-financial-services)
5. [Third-Party API Integrations](#5-third-party-api-integrations)
6. [Security Implementation](#6-security-implementation)
7. [Deployment & Scaling Strategy](#7-deployment--scaling-strategy)
8. [Implementation Roadmap](#8-implementation-roadmap)

---

## 1. Architecture Overview

### System Architecture

The Kodo X backend follows a modern serverless architecture built on Supabase with the following components:

- **Database Layer**: PostgreSQL database hosted by Supabase
- **Authentication Layer**: Supabase Auth with custom phone authentication
- **Storage Layer**: Supabase Storage for documents and media
- **API Layer**: 
  - Supabase Edge Functions for custom business logic
  - RESTful endpoints for client-server communication
  - WebSockets for real-time updates
- **Integration Layer**: Secure connectors to third-party financial services
- **Notification System**: Push, SMS, and email notification services

### Technology Stack

- **Core Platform**: Supabase (PostgreSQL, Auth, Storage, Edge Functions)
- **Server Runtime**: Deno (for Edge Functions)
- **API Format**: REST with JSON
- **Real-time Updates**: Supabase Realtime
- **Monitoring**: Supabase Monitoring + Custom Logging
- **CI/CD**: GitHub Actions

### System Diagram

```
┌─────────────────┐     ┌─────────────────────────────────────┐
│                 │     │             Supabase                │
│  Mobile Client  │◄────┤  ┌─────────┐  ┌──────┐  ┌────────┐  │
│  (React Native) │     │  │PostgreSQL│  │ Auth │  │Storage │  │
│                 │────►│  └─────────┘  └──────┘  └────────┘  │
└─────────────────┘     │         ┌─────────────┐             │
                        │         │Edge Functions│             │
                        │         └─────────────┘             │
                        └─────────────────┬─────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────┐
│                   Third-Party Integrations                    │
│                                                              │
│  ┌──────────────┐  ┌──────────┐  ┌────────────┐  ┌────────┐  │
│  │  Flutterwave │  │ Chimoney │  │ Union54/   │  │ Twilio │  │
│  │  (Payments)  │  │ (Wallets)│  │ Chimoney   │  │ (SMS)  │  │
│  └──────────────┘  └──────────┘  │ (Cards)    │  └────────┘  │
│                                  └────────────┘              │
└──────────────────────────────────────────────────────────────┘
```

## 2. Database Schema

### Core Tables

#### Users & Authentication

```sql
-- Users table (extends Supabase auth.users)
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name TEXT NOT NULL,
  phone_number TEXT UNIQUE NOT NULL,
  email TEXT UNIQUE,
  wallet_tag TEXT UNIQUE,
  date_of_birth DATE,
  address_city TEXT,
  address_region TEXT,
  address_area TEXT,
  profile_photo_url TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_active BOOLEAN NOT NULL DEFAULT true,
  is_verified BOOLEAN NOT NULL DEFAULT false,
  verification_status TEXT DEFAULT 'pending' CHECK (verification_status IN ('pending', 'verified', 'rejected')),
  verification_date TIMESTAMPTZ,
  last_login TIMESTAMPTZ
);

-- KYC verification documents
CREATE TABLE public.verification_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  document_type TEXT NOT NULL CHECK (document_type IN ('id_front', 'id_back', 'passport', 'selfie')),
  document_url TEXT NOT NULL,
  verification_status TEXT NOT NULL DEFAULT 'pending' CHECK (verification_status IN ('pending', 'approved', 'rejected')),
  rejection_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Wallets & Balances

```sql
-- Wallets table
CREATE TABLE public.wallets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  currency TEXT NOT NULL CHECK (currency IN ('GMD', 'USD')),
  balance DECIMAL(18, 2) NOT NULL DEFAULT 0,
  available_balance DECIMAL(18, 2) NOT NULL DEFAULT 0,
  pending_balance DECIMAL(18, 2) NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(user_id, currency)
);

-- Virtual Cards
CREATE TABLE public.virtual_cards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  wallet_id UUID NOT NULL REFERENCES public.wallets(id) ON DELETE CASCADE,
  card_number TEXT,
  masked_card_number TEXT,
  expiry_date TEXT,
  cvv TEXT,
  cardholder_name TEXT NOT NULL,
  provider TEXT NOT NULL,
  provider_card_id TEXT,
  is_active BOOLEAN NOT NULL DEFAULT false,
  is_locked BOOLEAN NOT NULL DEFAULT false,
  monthly_fee DECIMAL(10, 2) NOT NULL,
  next_billing_date DATE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Card Settings
CREATE TABLE public.card_settings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES public.virtual_cards(id) ON DELETE CASCADE,
  online_payments BOOLEAN NOT NULL DEFAULT true,
  international_usage BOOLEAN NOT NULL DEFAULT true,
  contactless_payments BOOLEAN NOT NULL DEFAULT false,
  atm_withdrawals BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Transactions & Financial Operations

```sql
-- Transactions table (for all financial movements)
CREATE TABLE public.transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  transaction_type TEXT NOT NULL CHECK (transaction_type IN ('topup', 'send', 'receive', 'convert', 'card_payment', 'card_topup', 'bank_transfer', 'fee', 'refund')),
  source_type TEXT NOT NULL CHECK (source_type IN ('wallet', 'card', 'bank', 'mobile_money', 'external')),
  source_id UUID,
  destination_type TEXT NOT NULL CHECK (destination_type IN ('wallet', 'card', 'bank', 'mobile_money', 'external')),
  destination_id UUID,
  amount DECIMAL(18, 2) NOT NULL,
  fee DECIMAL(10, 2) NOT NULL DEFAULT 0,
  currency TEXT NOT NULL CHECK (currency IN ('GMD', 'USD')),
  exchange_rate DECIMAL(10, 5),
  status TEXT NOT NULL CHECK (status IN ('pending', 'completed', 'failed', 'cancelled')),
  reference TEXT,
  description TEXT,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Transaction details for specific transaction types
CREATE TABLE public.transaction_details (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL REFERENCES public.transactions(id) ON DELETE CASCADE,
  provider TEXT, -- For mobile money or bank providers
  recipient_name TEXT,
  recipient_phone TEXT,
  recipient_wallet_tag TEXT,
  recipient_bank_account TEXT,
  recipient_bank_name TEXT,
  receipt_url TEXT, -- For bank transfer receipts
  note TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Currency conversion records
CREATE TABLE public.currency_conversions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  transaction_id UUID NOT NULL REFERENCES public.transactions(id) ON DELETE CASCADE,
  from_currency TEXT NOT NULL CHECK (from_currency IN ('GMD', 'USD')),
  to_currency TEXT NOT NULL CHECK (to_currency IN ('GMD', 'USD')),
  from_amount DECIMAL(18, 2) NOT NULL,
  to_amount DECIMAL(18, 2) NOT NULL,
  exchange_rate DECIMAL(10, 5) NOT NULL,
  fee DECIMAL(10, 2) NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Card transactions (specific to virtual card usage)
CREATE TABLE public.card_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES public.virtual_cards(id) ON DELETE CASCADE,
  transaction_id UUID NOT NULL REFERENCES public.transactions(id) ON DELETE CASCADE,
  merchant_name TEXT NOT NULL,
  merchant_category TEXT,
  amount DECIMAL(18, 2) NOT NULL,
  currency TEXT NOT NULL DEFAULT 'USD',
  status TEXT NOT NULL CHECK (status IN ('pending', 'completed', 'failed', 'declined')),
  provider_transaction_id TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 3. Authentication & User Management

### Authentication Flow

1. **Phone Number Registration**:
   - User enters phone number and creates a PIN
   - System sends OTP via Supabase Auth (with Twilio integration)
   - User verifies OTP to create account
   - Profile record created in `profiles` table

2. **Login Process**:
   - User enters phone number and PIN
   - System sends OTP for verification
   - Upon successful verification, JWT token issued
   - Last login timestamp updated

3. **KYC Verification**:
   - New users upload ID documents and selfie
   - Documents stored in Supabase Storage
   - References saved in `verification_documents` table
   - Admin reviews documents via dashboard
   - User notified of verification status

### Implementation with Supabase Auth

```typescript
// Example: User registration with phone auth
async function registerUser(phoneNumber, pin) {
  try {
    // 1. Create user in Supabase Auth
    const { data, error } = await supabase.auth.signUp({
      phone: phoneNumber,
      password: pin, // Hashed by Supabase
    });
    
    if (error) throw error;
    
    // 2. Send OTP via Twilio integration
    const { error: otpError } = await supabase.auth.signInWithOtp({
      phone: phoneNumber,
    });
    
    if (otpError) throw otpError;
    
    // 3. Create profile record (will be completed after OTP verification)
    return { success: true, userId: data.user.id };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

// Example: OTP verification
async function verifyOTP(phoneNumber, otp) {
  try {
    const { data, error } = await supabase.auth.verifyOtp({
      phone: phoneNumber,
      token: otp,
      type: 'sms',
    });
    
    if (error) throw error;
    
    // Update profile with verified status
    return { success: true, user: data.user };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

### User Profile Management

- **Profile Creation**: Automatically create profile after successful registration
- **Profile Updates**: Allow users to update personal information with verification
- **KYC Process**: Implement document upload and verification workflow
- **Security**: Implement PIN changes with OTP verification

## 4. Core Financial Services

### Wallet Management

1. **Wallet Creation**:
   - Automatically create GMD and USD wallets for new users
   - Initialize with zero balance
   - Assign unique wallet tags

2. **Balance Management**:
   - Track available, pending, and total balances
   - Implement transaction-based balance updates
   - Provide real-time balance information

3. **Transaction Processing**:
   - Implement atomic transactions with database triggers
   - Use PostgreSQL functions for complex operations
   - Maintain transaction history with detailed metadata

### Money Transfer Services

1. **Internal Transfers**:
   - User-to-user transfers within Kodo X
   - Support for wallet tag and phone number lookup
   - Real-time transfer notifications

2. **Mobile Money Integration**:
   - Connect with QMoney, AfriMoney, and Wave
   - Support for deposits and withdrawals
   - Transaction status tracking and reconciliation

3. **Bank Transfers**:
   - Integration with local Gambian banks
   - Support for deposits via bank transfer
   - Manual verification process for bank deposits

### Virtual Card Services

1. **Card Issuance**:
   - Integration with Union54 or Chimoney for virtual card issuance
   - Secure card data storage and display
   - Card activation and subscription management

2. **Card Management**:
   - Lock/unlock functionality
   - Transaction limits and controls
   - Card settings management

3. **Card Transactions**:
   - Process card transactions via provider webhooks
   - Categorize and store transaction data
   - Provide transaction history and analytics

### Currency Conversion

1. **Exchange Rate Management**:
   - Real-time or near-real-time exchange rates
   - Rate markup configuration for revenue
   - Historical rate tracking

2. **Conversion Process**:
   - GMD to USD and USD to GMD conversions
   - Fee calculation and application
   - Transaction records for all conversions

## 5. Third-Party API Integrations

### Payment Processing

#### Flutterwave Integration

**Purpose**: Mobile money processing, bank transfers, and payment collection

**Integration Points**:
- **Mobile Money Top-ups**: Process mobile money deposits to user wallets
- **Bank Transfers**: Process bank transfer deposits and withdrawals
- **Payment Collection**: Accept payments from various sources

**API Implementation**:
```typescript
// Example: Process mobile money top-up via Flutterwave
async function processMobileMoneyTopUp(userId, amount, phoneNumber, provider) {
  try {
    // 1. Create Flutterwave charge request
    const payload = {
      tx_ref: `KODOX-MM-${Date.now()}`,
      amount: amount,
      currency: "GMD",
      payment_type: "mobilemoneygm",
      country: "GM",
      email: userEmail,
      phone_number: phoneNumber,
      network: provider, // e.g., "QMoney", "AfriMoney"
      fullname: userName,
      client_ip: "0.0.0.0",
      device_fingerprint: "device_fingerprint",
      meta: {
        user_id: userId
      }
    };
    
    const response = await fetch('https://api.flutterwave.com/v3/charges?type=mobile_money_gambia', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${FLUTTERWAVE_SECRET_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(payload)
    });
    
    const data = await response.json();
    
    // 2. Create pending transaction in database
    const { data: transaction, error } = await supabase
      .from('transactions')
      .insert({
        user_id: userId,
        transaction_type: 'topup',
        source_type: 'mobile_money',
        destination_type: 'wallet',
        amount: amount,
        currency: 'GMD',
        status: 'pending',
        reference: data.data.tx_ref,
        metadata: data.data
      })
      .select()
      .single();
    
    if (error) throw error;
    
    return { success: true, transaction, flutterwaveResponse: data };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

#### Chimoney Integration

**Purpose**: Virtual card issuance, wallet management, and cross-border payments

**Integration Points**:
- **Virtual Cards**: Issue and manage virtual Visa/Mastercard cards
- **Multi-currency Wallets**: Alternative wallet provider for USD
- **Cross-border Payments**: International payment processing

**API Implementation**:
```typescript
// Example: Issue virtual card via Chimoney
async function issueVirtualCard(userId, cardholderName) {
  try {
    // 1. Request card issuance from Chimoney
    const payload = {
      name: cardholderName,
      email: userEmail,
      phone: userPhone,
      valueInUSD: 0, // Initial zero balance
      issueCard: true,
      cardType: "VIRTUAL"
    };
    
    const response = await fetch('https://api.chimoney.io/v0.2/cards/issue', {
      method: 'POST',
      headers: {
        'X-API-KEY': CHIMONEY_API_KEY,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(payload)
    });
    
    const data = await response.json();
    
    if (!data.success) throw new Error(data.message || 'Card issuance failed');
    
    // 2. Store card details in database
    const { data: card, error } = await supabase
      .from('virtual_cards')
      .insert({
        user_id: userId,
        wallet_id: userUsdWalletId,
        card_number: data.data.cardNumber,
        masked_card_number: data.data.maskedCardNumber,
        expiry_date: data.data.expiryDate,
        cvv: data.data.cvv,
        cardholder_name: cardholderName,
        provider: 'Chimoney',
        provider_card_id: data.data.id,
        is_active: true,
        is_locked: false,
        monthly_fee: 150, // GMD 150 monthly fee
        next_billing_date: new Date(Date.now() + 30*24*60*60*1000) // 30 days from now
      })
      .select()
      .single();
    
    if (error) throw error;
    
    return { success: true, card };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

### SMS and Notifications

#### Twilio Integration

**Purpose**: OTP delivery, transaction notifications, and account alerts

**Integration Points**:
- **OTP Delivery**: Send verification codes for authentication
- **Transaction Alerts**: Notify users of account activity
- **Security Alerts**: Send security-related notifications

**API Implementation**:
```typescript
// Example: Send OTP via Twilio
async function sendOTP(phoneNumber, otp) {
  try {
    const response = await fetch('https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages.json', {
      method: 'POST',
      headers: {
        'Authorization': `Basic ${btoa(`${TWILIO_ACCOUNT_SID}:${TWILIO_AUTH_TOKEN}`)}`,
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        To: phoneNumber,
        From: TWILIO_PHONE_NUMBER,
        Body: `Your Kodo X verification code is: ${otp}. Valid for 10 minutes.`
      })
    });
    
    const data = await response.json();
    return { success: true, messageId: data.sid };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

### KYC and Identity Verification

#### Smile Identity Integration (Optional)

**Purpose**: Automated KYC verification and identity confirmation

**Integration Points**:
- **Document Verification**: Validate ID documents
- **Facial Recognition**: Match selfie to ID photo
- **Anti-fraud Checks**: Detect fraudulent documents

**API Implementation**:
```typescript
// Example: Verify ID document with Smile Identity
async function verifyIdentity(userId, idFrontUrl, idBackUrl, selfieUrl, idType) {
  try {
    // 1. Prepare verification request
    const payload = {
      partner_id: SMILE_PARTNER_ID,
      job_id: `KODOX-KYC-${userId}`,
      job_type: 5, // Document verification with selfie
      id_info: {
        country: "GM",
        id_type: idType === 'ID' ? 'NATIONAL_ID' : 'PASSPORT',
        first_name: userFirstName,
        last_name: userLastName,
        dob: userDateOfBirth,
        phone_number: userPhone
      },
      options: {
        return_job_status: true,
        return_history: true,
        return_images: true
      },
      images: {
        id_card: idFrontUrl,
        id_card_back: idType === 'ID' ? idBackUrl : null,
        selfie: selfieUrl
      }
    };
    
    // 2. Send verification request
    const response = await fetch('https://api.smileidentity.com/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Basic ${btoa(`${SMILE_API_KEY}:`)}`
      },
      body: JSON.stringify(payload)
    });
    
    const data = await response.json();
    
    // 3. Update verification status in database
    const { error } = await supabase
      .from('verification_documents')
      .update({
        verification_status: data.result.ResultCode === "1001" ? 'approved' : 'rejected',
        rejection_reason: data.result.ResultCode !== "1001" ? data.result.ResultText : null
      })
      .eq('user_id', userId);
    
    if (error) throw error;
    
    return { success: true, verificationResult: data.result };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

## 6. Security Implementation

### Data Protection

1. **Encryption**:
   - Encrypt sensitive data at rest using PostgreSQL column encryption
   - Use TLS/SSL for all data in transit
   - Implement secure key management

2. **PCI Compliance**:
   - Store card data with PCI-compliant providers only
   - Implement tokenization for card references
   - Regular security audits and compliance checks

### API Security

1. **Authentication**:
   - JWT-based authentication for all API requests
   - Short-lived tokens with refresh mechanism
   - Role-based access control (RBAC)

2. **Request Validation**:
   - Input validation on all API endpoints
   - Rate limiting to prevent abuse
   - Request signing for critical operations

### Database Security

1. **Row-Level Security (RLS)**:
   - Implement RLS policies for all tables
   - Ensure users can only access their own data
   - Admin-specific policies for management functions

```sql
-- Example RLS policies
-- Users can only read their own profile
CREATE POLICY "Users can read own profile" ON public.profiles
  FOR SELECT USING (auth.uid() = id);

-- Users can only update their own profile
CREATE POLICY "Users can update own profile" ON public.profiles
  FOR UPDATE USING (auth.uid() = id);

-- Users can only view their own wallets
CREATE POLICY "Users can view own wallets" ON public.wallets
  FOR SELECT USING (auth.uid() = user_id);

-- Users can only view their own transactions
CREATE POLICY "Users can view own transactions" ON public.transactions
  FOR SELECT USING (auth.uid() = user_id);
```

2. **Audit Logging**:
   - Track all sensitive operations
   - Log user actions for compliance
   - Implement database triggers for automatic logging

```sql
-- Example audit logging trigger
CREATE OR REPLACE FUNCTION log_profile_changes()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_logs (
    user_id,
    action,
    entity_type,
    entity_id,
    old_values,
    new_values
  ) VALUES (
    auth.uid(),
    TG_OP,
    'profile',
    NEW.id,
    row_to_json(OLD),
    row_to_json(NEW)
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER profile_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON public.profiles
FOR EACH ROW EXECUTE FUNCTION log_profile_changes();
```

## 7. Deployment & Scaling Strategy

### Infrastructure

1. **Supabase Project**:
   - Production environment on Supabase Pro plan
   - Staging environment for testing
   - Development environment for ongoing development

2. **Edge Functions Deployment**:
   - Automated deployment via GitHub Actions
   - Version control for all functions
   - Environment-specific configurations

### Scaling Strategy

1. **Database Scaling**:
   - Implement efficient indexing for frequently queried columns
   - Use connection pooling for optimal performance
   - Regular performance monitoring and optimization

2. **API Scaling**:
   - Stateless Edge Functions for horizontal scaling
   - Implement caching for frequently accessed data
   - Use background processing for heavy operations

3. **Storage Scaling**:
   - Implement file size limits and validation
   - Use CDN for frequently accessed files
   - Implement lifecycle policies for old files

### Monitoring and Maintenance

1. **Performance Monitoring**:
   - Track API response times and error rates
   - Monitor database query performance
   - Set up alerts for performance degradation

2. **Error Tracking**:
   - Implement structured error logging
   - Set up error notifications
   - Regular error review and resolution

3. **Backup and Disaster Recovery**:
   - Regular database backups
   - Point-in-time recovery capability
   - Disaster recovery plan and testing

## 8. Implementation Roadmap

### Phase 1: Core Infrastructure (Weeks 1-2)

- Set up Supabase project environments
- Create database schema with RLS policies
- Implement basic authentication flow
- Configure storage buckets and policies

**Deliverables**:
- Functional Supabase project with schema
- Basic authentication API endpoints
- Storage configuration for user documents

### Phase 2: User Management & KYC (Weeks 3-4)

- Implement phone authentication with OTP
- Create user profile management
- Build KYC document upload and verification flow
- Set up admin dashboard for KYC review

**Deliverables**:
- Complete user registration and login flow
- KYC document upload and verification system
- Admin interface for KYC approval

### Phase 3: Wallet & Basic Transactions (Weeks 5-6)

- Implement wallet creation and management
- Build internal transfer functionality
- Create transaction history and reporting
- Implement basic analytics

**Deliverables**:
- Functional GMD and USD wallets
- Internal transfer capability
- Transaction history API
- Basic spending analytics

### Phase 4: Payment Integrations (Weeks 7-8)

- Integrate Flutterwave for mobile money
- Implement bank transfer processing
- Build top-up and withdrawal flows
- Create payment reconciliation system

**Deliverables**:
- Mobile money deposit and withdrawal
- Bank transfer processing
- Complete top-up flow
- Payment webhooks and callbacks

### Phase 5: Virtual Cards (Weeks 9-10)

- Integrate Chimoney or Union54 for virtual cards
- Implement card issuance and activation
- Build card management features
- Create card transaction processing

**Deliverables**:
- Virtual card issuance and activation
- Card management interface
- Card transaction processing
- Card settings and controls

### Phase 6: Advanced Features & Optimization (Weeks 11-12)

- Implement currency conversion
- Build scheduled payments
- Create advanced analytics and insights
- Optimize performance and security

**Deliverables**:
- Currency conversion functionality
- Scheduled payment system
- Enhanced analytics and insights
- Performance and security improvements

## Comments

This backend architecture and API integration plan provides a comprehensive roadmap for building the Kodo X fintech platform. By leveraging Supabase's powerful features and integrating with key financial service providers like Flutterwave and Chimoney, we can create a robust, secure, and scalable system that meets all the requirements of a modern mobile fintech application.
