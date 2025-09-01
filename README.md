# One Trader Feature/Prompt SDK

## Overview
This SDK allows developers to create custom features and prompts for the Mr One AI Trading Assistant. Each feature follows a standardized structure to ensure consistency and easy integration.

## Feature Structure

### 1. Feature Manifest (`feature.json`)
```json
{
  "id": "unique-feature-id",
  "version": "1.0.0",
  "name": "Feature Display Name",
  "description": "Brief description of what this feature does",
  "author": {
    "name": "Developer Name",
    "github": "github-username",
    "contact": "email@example.com"
  },
  "prompt": {
    "icon": "🎯",
    "text": "Visible prompt text",
    "category": "trading|analysis|portfolio|utility",
    "color": "from-blue-500 to-cyan-500"
  },
  "api": {
    "endpoint": "/api/custom-feature",
    "method": "GET|POST",
    "requiresAuth": false,
    "requiresWallet": false,
    "parameters": []
  },
  "response": {
    "type": "data|trade|batch|analysis",
    "handler": "customHandler",
    "format": "json|markdown|html"
  },
  "permissions": {
    "readBalance": false,
    "executeTrade": false,
    "accessPortfolio": false
  },
  "testing": {
    "mockData": true,
    "testEndpoint": "/api/test/custom-feature"
  }
}
```

### 2. API Handler (`api/[feature-name]/route.js`)
```javascript
import { NextResponse } from 'next/server';
import { FeatureSDK } from '@/lib/feature-sdk';

// Initialize SDK
const sdk = new FeatureSDK({
  id: 'unique-feature-id',
  version: '1.0.0'
});

export async function GET(request) {
  try {
    // Validate request using SDK
    const validation = await sdk.validateRequest(request);
    if (!validation.valid) {
      return sdk.errorResponse(validation.error, 400);
    }
    
    // Get parameters
    const params = sdk.getParameters(request);
    
    // Your custom logic here
    const data = await yourCustomLogic(params);
    
    // Format response using SDK
    return sdk.successResponse({
      data,
      message: 'Feature executed successfully',
      metadata: {
        timestamp: new Date().toISOString(),
        cached: false
      }
    });
    
  } catch (error) {
    return sdk.errorResponse(error.message, 500);
  }
}

export async function POST(request) {
  try {
    const body = await sdk.parseBody(request);
    
    // Check permissions if needed
    if (sdk.requiresWallet() && !body.userPublicKey) {
      return sdk.errorResponse('Wallet connection required', 401);
    }
    
    // Process request
    const result = await processRequest(body);
    
    // Return formatted response
    return sdk.successResponse(result);
    
  } catch (error) {
    return sdk.errorResponse(error.message, 500);
  }
}
```

### 3. Response Handler (`handlers/[feature-name].js`)
```javascript
export const customHandler = {
  // Format the API response for display in chat
  formatResponse: (data) => {
    return {
      message: formatMessage(data),
      data: data.data,
      intent: data.intent || null,
      actions: data.actions || []
    };
  },
  
  // Format message for chat display
  formatMessage: (data) => {
    let message = `## ${data.title}\n\n`;
    
    // Add your custom formatting
    if (data.summary) {
      message += `**Summary:** ${data.summary}\n\n`;
    }
    
    if (data.details) {
      data.details.forEach(detail => {
        message += `• ${detail}\n`;
      });
    }
    
    return message;
  },
  
  // Handle any special actions
  handleActions: (actions, context) => {
    actions.forEach(action => {
      switch (action.type) {
        case 'trade':
          context.openTradeModal(action.intent);
          break;
        case 'alert':
          context.showAlert(action.message);
          break;
        default:
          console.log('Unknown action:', action);
      }
    });
  }
};
```

## SDK Core Library

### `lib/feature-sdk.js`
```javascript
export class FeatureSDK {
  constructor(config) {
    this.config = config;
    this.validators = new Validators();
    this.formatters = new Formatters();
  }
  
  // Request validation
  async validateRequest(request) {
    const checks = [
      this.validators.checkRateLimit(request),
      this.validators.checkAPIKey(request),
      this.validators.checkParameters(request, this.config)
    ];
    
    const results = await Promise.all(checks);
    const failed = results.find(r => !r.valid);
    
    return failed || { valid: true };
  }
  
  // Parameter extraction
  getParameters(request) {
    const url = new URL(request.url);
    const params = {};
    
    for (const [key, value] of url.searchParams) {
      params[key] = value;
    }
    
    return params;
  }
  
  // Body parsing
  async parseBody(request) {
    try {
      return await request.json();
    } catch {
      return {};
    }
  }
  
  // Response helpers
  successResponse(data, status = 200) {
    return NextResponse.json({
      success: true,
      ...data,
      featureId: this.config.id,
      version: this.config.version
    }, { status });
  }
  
  errorResponse(message, status = 400) {
    return NextResponse.json({
      success: false,
      error: message,
      featureId: this.config.id
    }, { status });
  }
  
  // Permission checks
  requiresWallet() {
    return this.config.api?.requiresWallet || false;
  }
  
  requiresAuth() {
    return this.config.api?.requiresAuth || false;
  }
  
  // Data formatting
  formatTradeIntent(base, quote, amount, type = 'BUY') {
    return {
      type,
      base,
      quote,
      amount: amount.toString(),
      amountIs: type === 'BUY' ? 'QUOTE' : 'BASE',
      slippageBps: 300,
      timestamp: new Date().toISOString()
    };
  }
  
  // External API helpers
  async fetchWithCache(url, options = {}, ttl = 300) {
    const cacheKey = `${this.config.id}:${url}`;
    const cached = await this.getCache(cacheKey);
    
    if (cached && cached.expires > Date.now()) {
      return cached.data;
    }
    
    const response = await fetch(url, options);
    const data = await response.json();
    
    await this.setCache(cacheKey, data, ttl);
    return data;
  }
  
  // Cache operations
  async getCache(key) {
    // Implement your cache logic (Redis, memory, etc.)
    return null;
  }
  
  async setCache(key, data, ttl) {
    // Implement your cache logic
    return true;
  }
}

// Validators
class Validators {
  async checkRateLimit(request) {
    // Implement rate limiting
    return { valid: true };
  }
  
  async checkAPIKey(request) {
    // Implement API key validation if needed
    return { valid: true };
  }
  
  async checkParameters(request, config) {
    // Validate required parameters
    return { valid: true };
  }
}

// Formatters
class Formatters {
  currency(amount, decimals = 2) {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals
    }).format(amount);
  }
  
  percentage(value, decimals = 2) {
    return `${value > 0 ? '+' : ''}${value.toFixed(decimals)}%`;
  }
  
  number(value, decimals = 2) {
    return value.toLocaleString(undefined, {
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals
    });
  }
}
```

## Feature Categories

### 1. Trading Features
Features that execute or prepare trades.

**Example: DCA Bot**
```javascript
{
  "id": "dca-bot",
  "name": "DCA Trading Bot",
  "prompt": {
    "icon": "🤖",
    "text": "Setup DCA for SOL",
    "category": "trading"
  },
  "api": {
    "endpoint": "/api/dca-bot",
    "method": "POST",
    "requiresWallet": true
  },
  "response": {
    "type": "trade",
    "handler": "dcaHandler"
  }
}
```

### 2. Analysis Features
Features that analyze data and provide insights.

**Example: Whale Tracker**
```javascript
{
  "id": "whale-tracker",
  "name": "Whale Movement Tracker",
  "prompt": {
    "icon": "🐋",
    "text": "Track whale movements",
    "category": "analysis"
  },
  "api": {
    "endpoint": "/api/whale-tracker",
    "method": "GET"
  },
  "response": {
    "type": "data",
    "handler": "whaleHandler"
  }
}
```

### 3. Portfolio Features
Features that interact with user's portfolio.

**Example: Rebalancer**
```javascript
{
  "id": "portfolio-rebalancer",
  "name": "Portfolio Rebalancer",
  "prompt": {
    "icon": "⚖️",
    "text": "Rebalance my portfolio",
    "category": "portfolio"
  },
  "api": {
    "endpoint": "/api/rebalance",
    "method": "POST",
    "requiresWallet": true
  },
  "permissions": {
    "readBalance": true,
    "executeTrade": true
  }
}
```

### 4. Utility Features
General utility features.

**Example: Gas Tracker**
```javascript
{
  "id": "gas-tracker",
  "name": "Solana Gas Tracker",
  "prompt": {
    "icon": "⛽",
    "text": "Current gas prices",
    "category": "utility"
  },
  "api": {
    "endpoint": "/api/gas-tracker",
    "method": "GET"
  }
}
```

## Submission Process

### 1. Development
```bash
# Clone the SDK template
git clone https://github.com/your-org/feature-sdk-template my-feature

# Install dependencies
cd my-feature
npm install

# Develop your feature
npm run dev
```

### 2. Testing
```bash
# Run tests
npm test

# Test with mock data
npm run test:mock

# Integration test
npm run test:integration
```

### 3. Validation
```bash
# Validate feature structure
npm run validate

# Check compliance
npm run lint
```

### 4. Submission
```javascript
// submit-feature.js
import { FeatureSubmitter } from '@/lib/feature-submitter';

const submitter = new FeatureSubmitter({
  apiKey: process.env.SUBMISSION_API_KEY
});

await submitter.submit({
  manifestPath: './feature.json',
  codePath: './api',
  handlerPath: './handlers',
  testResults: './test-results.json'
});
```

## Integration Guide

### For Platform Owners
```javascript
// feature-registry.js
export class FeatureRegistry {
  constructor() {
    this.features = new Map();
  }
  
  async register(featureManifest) {
    // Validate manifest
    const validation = await this.validate(featureManifest);
    if (!validation.valid) {
      throw new Error(validation.error);
    }
    
    // Register feature
    this.features.set(featureManifest.id, featureManifest);
    
    // Setup routes
    await this.setupRoutes(featureManifest);
    
    // Add to UI
    await this.addToUI(featureManifest);
    
    return true;
  }
  
  async validate(manifest) {
    // Check required fields
    // Validate permissions
    // Test endpoints
    // Security scan
    return { valid: true };
  }
}
```

### For Feature Developers
```javascript
// Example: Custom DEX Aggregator Feature
export default class DexAggregatorFeature {
  constructor(sdk) {
    this.sdk = sdk;
  }
  
  async execute(params) {
    // Fetch prices from multiple DEXs
    const prices = await Promise.all([
      this.fetchJupiter(params),
      this.fetchOrcaDex(params),
      this.fetchRaydium(params)
    ]);
    
    // Find best price
    const best = this.findBestPrice(prices);
    
    // Return formatted response
    return this.sdk.successResponse({
      data: {
        bestPrice: best,
        allPrices: prices,
        recommendation: this.getRecommendation(best)
      },
      intent: best.tradeIntent,
      message: 'Found best price across DEXs'
    });
  }
}
```

## Security Guidelines

1. **Input Validation**: Always validate user inputs
2. **Rate Limiting**: Implement rate limiting
3. **Authentication**: Use proper authentication when needed
4. **Sanitization**: Sanitize all outputs
5. **Error Handling**: Never expose sensitive information in errors
6. **Permissions**: Request minimum necessary permissions

## Best Practices

1. **Performance**: Cache expensive operations
2. **Error Handling**: Provide meaningful error messages
3. **Documentation**: Document all parameters and responses
4. **Testing**: Include comprehensive tests
5. **Versioning**: Use semantic versioning
6. **Backwards Compatibility**: Maintain compatibility when updating

## Support

- Documentation: https://docs.your-platform.com/sdk
- Examples: https://github.com/your-org/feature-examples
- Discord: https://discord.gg/your-server
- Submit Issues: https://github.com/your-org/sdk/issues
