# Project Overview

This is a Next.js application built with TypeScript and styled using Tailwind CSS. The project follows modern React patterns and Next.js best practices, optimized for deployment on AWS Amplify.

## Tech Stack

- **Framework**: Next.js 14+ (App Router)
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **Deployment**: AWS Amplify
- **Package Manager**: pnpm

## Coding Standards

### TypeScript
- Use explicit type annotations for function parameters and return types
- Prefer interfaces over types for object shapes
- Use proper TypeScript generics where applicable
- Avoid using `any` type; use `unknown` if type is truly unknown
- Export types/interfaces that are used across multiple files

### React/Next.js Patterns
- Use functional components with TypeScript interfaces for props
- Implement proper error boundaries for error handling
- Use Next.js App Router conventions (page.tsx, layout.tsx, loading.tsx, error.tsx)
- Leverage Server Components by default, use Client Components only when needed
- Use `use client` directive at the top of files that need client-side interactivity

### Component Structure
```typescript
// Example component structure
interface ComponentNameProps {
  title: string;
  children?: React.ReactNode;
}

export default function ComponentName({ title, children }: ComponentNameProps) {
  return (
    <div className="...">
      <h2>{title}</h2>
      {children}
    </div>
  );
}
```

### Tailwind CSS Guidelines
- Use Tailwind utility classes exclusively for styling
- Follow mobile-first responsive design (use sm:, md:, lg:, xl: prefixes)
- Utilize Tailwind's color palette and spacing scale for consistency
- Group related utilities together for readability
- Common patterns:
  - Flexbox: `flex items-center justify-between`
  - Grid: `grid grid-cols-1 md:grid-cols-2 gap-4`
  - Spacing: Use consistent spacing scale (p-4, m-2, gap-6, etc.)
  - Typography: `text-base font-medium text-gray-900 dark:text-gray-100`

### File Naming Conventions
- Components: PascalCase (e.g., `UserProfile.tsx`)
- Utilities/Hooks: camelCase (e.g., `useAuth.ts`, `formatDate.ts`)
- Types/Interfaces: PascalCase with descriptive names (e.g., `UserProfile.types.ts`)
- Use `.tsx` for files with JSX, `.ts` for pure TypeScript files

## State Management
- Use React's built-in hooks (useState, useReducer) for local state
- For server state, leverage Next.js data fetching (Server Components)
- Consider React Context for cross-component state without external libraries
- Use URL state (searchParams) for shareable application state

## Data Fetching
- Prefer Server Components for data fetching when possible
- Use `async`/`await` in Server Components
- Implement proper loading and error states
- For client-side fetching, create custom hooks with proper TypeScript types

## Performance Considerations
- Use Next.js Image component for optimized images
- Implement proper code splitting with dynamic imports when needed
- Minimize client-side JavaScript by leveraging Server Components
- Use Tailwind's purge configuration for production builds

## Accessibility
- Always include proper ARIA labels
- Ensure keyboard navigation works properly
- Use semantic HTML elements
- Maintain proper heading hierarchy
- Include alt text for all images

## Common Utilities

### Custom Tailwind Class Helper
```typescript
// lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

## Environment Variables
- Use `.env.local` for local development
- Prefix client-side variables with `NEXT_PUBLIC_`
- Access server-side variables only in Server Components or API routes
- Configure environment variables in AWS Amplify Console under App settings > Environment variables
- Never commit sensitive environment variables
- Use AWS Systems Manager Parameter Store for sensitive values when needed

## AWS Amplify Configuration

### Build Settings (amplify.yml)
```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
      - .next/cache/**/*
```

### Next.js Configuration for Amplify
```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  images: {
    unoptimized: true, // Required for static export if using SSG
    // Or configure domains for external images
    domains: ['your-domain.com'],
  },
  // Enable if using ISR
  experimental: {
    isrMemoryCacheSize: 0, // Recommended for Amplify
  },
}

module.exports = nextConfig
```

### Deployment Considerations
- Amplify supports SSR, SSG, and ISR for Next.js
- For SSR: Ensure your app works with Node.js 18.x runtime
- For static export: Use `output: 'export'` in next.config.js
- Monitor build times (Amplify has build time limits)
- Use Amplify's branch preview feature for PR reviews

## Performance Optimizations for Amplify
- Enable Amplify's built-in performance monitoring
- Use CloudFront distribution provided by Amplify
- Implement proper cache headers for static assets
- Consider using AWS CloudWatch for custom metrics
- Optimize images using Next.js Image component with proper sizing

## Testing Approach
- Write unit tests for utility functions
- Component testing focuses on user interactions
- Use TypeScript for type safety in tests
- Test accessibility requirements

## Key Commands
```bash
# Development
npm run dev

# Build (local testing of Amplify build)
npm run build

# Production (local)
npm start

# Type checking
npm run type-check

# Linting
npm run lint

# Amplify CLI commands (if using)
amplify init
amplify push
amplify publish
```

## AWS Amplify Integration

### Backend Services (if using Amplify Backend)
- Authentication: Use Amplify Auth with Cognito
- API: GraphQL with AppSync or REST with API Gateway
- Storage: S3 for file uploads
- Database: DynamoDB or RDS via AppSync

### Frontend Integration
```typescript
// Example: Amplify configuration
import { Amplify } from 'aws-amplify';
import config from '@/aws-exports';

Amplify.configure(config);
```

```typescript
// Using Amplify API
import { API } from 'aws-amplify';
const result = await API.post('ebay-api', '/auctions', { body: data });
```

### Common Amplify Patterns
- Use Server Components for initial data fetching
- Implement proper error boundaries for API failures
- Handle authentication states properly
- Use Amplify DataStore for offline capabilities if needed

## Deployment Best Practices
- Set up automatic deployments from your Git branch
- Configure proper build settings in amplify.yml
- Use Amplify Console for managing environment variables
- Set up custom domains and SSL certificates
- Monitor deployments using Amplify Console logs
- Configure proper headers for security (CSP, CORS)
- Use Amplify's Web Application Firewall (WAF) if needed

## Important Notes
- Always validate and sanitize user inputs
- Handle loading and error states gracefully
- Follow Next.js caching strategies (be aware of Amplify's caching behavior)
- Keep components small and focused
- Use proper TypeScript types for all data structures
- Implement proper SEO with Next.js Metadata API
- Consider Amplify's build limits (memory, timeout, artifact size)
- Monitor costs associated with SSR functions in Amplify
- Use Amplify's built-in monitoring and alerting features

## Troubleshooting Amplify Deployments
- Check build logs in Amplify Console for errors
- Verify all environment variables are set correctly
- Ensure package.json scripts match Amplify build commands
- Check Next.js version compatibility with Amplify
- Verify node version compatibility (Amplify uses Node.js 18.x)
- For SSR issues, check CloudWatch logs for Lambda functions

## When Providing Code
- Always use TypeScript with proper type annotations
- Include all necessary imports
- Use Tailwind classes for all styling
- Follow the established project structure
- Consider both mobile and desktop viewports
- Handle edge cases and error scenarios