# B2C Commerce Cloud — Storefront Architecture Assessment

**Author:** Luciano Diaz — Principal Solution Engineer at Salesforce

**Salesforce Credentials:**
- Certified B2C Commerce Developer
- Certified Data Cloud Consultant
- Certified Agentforce Specialist
- Certified AI Associate
- Certified Platform Foundations

**Date:** June 2026

---

## Document Objective

This document aims to provide a technical-functional assessment of the available implementation alternatives for Salesforce B2C Commerce Cloud in the context of an architecture evaluation for an organization seeking to modernize or evolve its digital commerce channel. Unlike a commercial comparison or a general overview of capabilities, this analysis focuses on explaining the architectural, functional, and operational implications of the three main storefront building approaches on B2C Commerce Cloud: Storefront Reference Architecture, PWA Kit / Composable Storefront, and a headless or composable approach at a technological level. The purpose is to deliver a clear foundation for understanding how each modality addresses the digital commerce experience, what level of flexibility it offers, which Salesforce components are involved, what their technical dependencies are, and what considerations must be evaluated in terms of time-to-market, maintainability, scalability, integration, and future evolution.

This document takes as its starting point the need to understand, in greater depth, the implementation options that Salesforce B2C Commerce Cloud offers to support a modern digital commerce strategy. In this regard, the assessment does not seek to define solely a technological preference, but rather to establish decision criteria that allow each alternative to be contrasted against business needs, the technological maturity of the current ecosystem, the capacity for integration with corporate systems, the expected experience for end users, and the operational model required to sustain the platform over time. For each approach, the role of the storefront, the relationship with B2C Commerce Cloud, the use of APIs, the separation between user experience and commerce logic, the degree of coupling with the platform, and the scenarios in which each alternative may be most suitable depending on the organization's context will be analyzed.

---

## 1. Storefront Reference Architecture (SFRA)

### What is SFRA

Storefront Reference Architecture (SFRA) is the official reference storefront provided by Salesforce as the foundation for building B2C Commerce Cloud storefronts. It is a server-side rendered application that runs entirely within the B2C Commerce Cloud platform infrastructure, meaning both the commerce logic and the presentation layer are tightly coupled and executed on Salesforce's servers.

SFRA replaced the previous reference architecture known as SiteGenesis, offering a more modular and maintainable codebase while preserving the same fundamental server-side rendering paradigm. It serves as both a functional starting point for new implementations and a demonstration of platform best practices.

### Architectural Model

SFRA follows a Model-View-Controller (MVC) pattern executed server-side on the B2C Commerce Cloud application server:

- **Controllers** are server-side JavaScript files that handle incoming HTTP requests, orchestrate business logic, and prepare data for rendering. They run on the Salesforce B2C Commerce script engine (Rhino-based) and have direct access to platform APIs for catalog, cart, checkout, customer, and order operations.

- **Models** are JavaScript objects that encapsulate and transform raw platform data into structured representations consumed by the view layer. They act as an abstraction between the platform's internal data model and what the templates need to render.

- **Views** are ISML (Internet Store Markup Language) templates — a proprietary Salesforce templating language that generates HTML on the server before sending the response to the browser. ISML supports loops, conditionals, template inclusion, and decorator patterns for layout composition.

### Cartridge System and Inheritance

The core extensibility mechanism in SFRA is the **cartridge path** — an ordered list of cartridges that determines how the platform resolves controllers, templates, static assets, and scripts at runtime. Each cartridge is essentially a self-contained module with a defined directory structure.

When a request arrives, the platform traverses the cartridge path from left to right, resolving the first matching controller or template it finds. This enables a layered override model:

- **Base cartridges** (`app_storefront_base`) contain the default SFRA implementation.
- **Custom cartridges** are placed earlier in the path and can override or extend any controller, template, or model from the base.
- **Plugin cartridges** provide optional functionality (e.g., `plugin_applepay`, `plugin_wishlist`) and are inserted between the custom and base cartridges.

This inheritance model allows teams to customize behavior without modifying the base code, which simplifies upgrade paths when Salesforce releases new versions of SFRA.

### Server-Side Rendering and Request Lifecycle

Every page request in SFRA follows this lifecycle:

1. The browser sends an HTTP request to B2C Commerce Cloud's CDN/edge layer.
2. If not cached, the request reaches the application server and is routed to the appropriate controller.
3. The controller executes business logic using platform script APIs (accessing catalogs, pricing, promotions, inventory, etc.).
4. The controller passes a view model to an ISML template.
5. The template renders the final HTML on the server.
6. The complete HTML response is returned to the browser.

Client-side JavaScript in SFRA is limited to progressive enhancement — form validation, AJAX calls for mini-cart updates, and UI interactions. The page structure and content are always resolved server-side.

### Relationship with Business Manager

SFRA is deeply integrated with Business Manager, the B2C Commerce Cloud back-office application. Merchants use Business Manager to manage:

- Product catalogs, categories, and pricing
- Promotions and campaigns
- Content slots and page layouts
- Site preferences and configuration
- Customer segments and A/B testing

Changes made in Business Manager are immediately reflected in the SFRA storefront without requiring code deployments. This tight coupling between the merchandising tool and the storefront is a defining characteristic of the SFRA model.

### Page Designer

Page Designer is the visual, no-code page editor built into Business Manager. It allows merchants to create, arrange, and publish pages by dragging and dropping components into regions — without writing code or requiring deployments.

In the SFRA context, Page Designer works with ISML-based component types. Developers build reusable page types and component types (banners, product carousels, hero images, content blocks) using ISML templates, and merchants assemble them visually in Business Manager. The architecture follows a three-tier hierarchy:

- **Pages** — top-level containers representing a full page (homepage, landing page, campaign page).
- **Regions** — logical content areas within a page that hold an ordered list of components.
- **Components** — the building blocks. Leaf components render content directly (images, text, product tiles). Layout components organize other components using grids or columns and can contain nested regions.

Merchants can schedule page publication, preview changes in context, and manage multiple page versions — all from Business Manager without developer intervention for content updates.

### Development Workflow

SFRA development relies on the following tools and processes:

- **Prophet Debugger / B2C Commerce IDE extensions** for VS Code provide code upload, debugging, and log streaming directly against sandbox instances.
- **Code deployment** is done by uploading cartridges to a sandbox via WebDAV or through CI/CD pipelines using the Salesforce B2C Commerce CLI (sfcc-ci).
- **Sandboxes** are individual development environments provided by the platform, each running a full instance of B2C Commerce Cloud.
- **ISML linting and compilation** happens at upload time — there is no local build step for templates.

The development feedback loop involves uploading code to a remote sandbox and testing against that environment, as there is no local runtime that replicates the B2C Commerce script engine.

### Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | B2C Commerce Cloud application server (Rhino-based JS engine) |
| Controllers | Server-side JavaScript (CommonJS modules) |
| Templates | ISML (Internet Store Markup Language) |
| Client-side | jQuery, vanilla JavaScript, AJAX |
| Styling | SCSS compiled to CSS (via build tools) |
| Build tools | Webpack (for client-side assets only) |
| Deployment | WebDAV upload / sfcc-ci CLI |
| Back-office | Business Manager |
| APIs consumed | Platform Script API (server-side, internal) |

### Code Examples

#### ISML Template — Page Layout with CSS and JS Includes

```isml
<isdecorate template="common/layout/page">

    <isscript>
        var assets = require('*/cartridge/scripts/assets');
        assets.addCss('/css/product/detail.css');
        assets.addJs('/js/product/detail.js');
    </isscript>

    <div class="container product-detail" data-pid="${product.id}">
        <div class="row">
            <div class="col-12 col-sm-6">
                <isinclude template="product/components/imageCarousel" />
            </div>
            <div class="col-12 col-sm-6">
                <h1 class="product-name">${product.productName}</h1>
                <div class="prices">
                    <isset name="price" value="${product.price}" scope="page" />
                    <isinclude template="product/components/pricing/main" />
                </div>
                <isif condition="${product.available}">
                    <button class="add-to-cart btn btn-primary">
                        ${Resource.msg('button.addtocart', 'common', null)}
                    </button>
                <iselse/>
                    <span class="out-of-stock">
                        ${Resource.msg('label.outofstock', 'common', null)}
                    </span>
                </isif>
            </div>
        </div>
    </div>

</isdecorate>
```

#### Controller — Server-Side JavaScript

```javascript
'use strict';

var server = require('server');
var cache = require('*/cartridge/scripts/middleware/cache');
var ProductFactory = require('*/cartridge/scripts/factories/product');

server.get('Show', cache.applyDefaultCache, function (req, res, next) {
    var params = req.querystring;
    var product = ProductFactory.get({ pid: params.pid });

    res.render('product/productDetails', {
        product: product
    });

    next();
});

module.exports = server.exports();
```

#### Client-Side JavaScript — AJAX Add to Cart

```javascript
'use strict';

var processInclude = require('base/util');

$(document).ready(function () {
    $('.add-to-cart').on('click', function (e) {
        e.preventDefault();
        var pid = $(this).closest('.product-detail').data('pid');

        $.ajax({
            url: '/on/demandware.store/Sites-SiteId-Site/default/Cart-AddProduct',
            method: 'POST',
            data: { pid: pid, quantity: 1 },
            success: function (data) {
                $('.minicart-quantity').text(data.quantityTotal);
                $('.minicart').trigger('count:update', data);
            }
        });
    });
});
```

### When to Use SFRA

SFRA is the standard approach for launching a storefront on B2C Commerce Cloud. It works with native JavaScript and the platform's own ecosystem (ISML, cartridges, Business Manager) without relying on external frontend libraries like React or Vue. It represents the first degree of implementation — the direct and complete path, with everything integrated.

### Strengths

- Comprehensive out-of-the-box solution with full commerce functionality from day one.
- Immediate merchandising capabilities through Business Manager — no code deployment needed for content, promotions, or catalog changes.
- Lower operational complexity — no separate frontend infrastructure to host, scale, or monitor.
- Mature and battle-tested at scale, with spectacular production sites across industries worldwide.

### Considerations

- The frontend is coupled to the platform, which simplifies operations but means that UX development is resolved within the B2C Commerce ecosystem rather than with external libraries like React or Vue.
- Development requires uploading code to remote sandboxes — there is no local runtime for the B2C Commerce script engine.
- The server-side rendering model means page transitions involve full round-trips to the server, unlike single-page application architectures.

### Official Documentation

| Resource | URL |
|----------|-----|
| B2C Commerce Cloud Developer Docs | https://developer.salesforce.com/docs/commerce/b2c-commerce/overview |
| SFRA Overview | https://developer.salesforce.com/docs/commerce/sfra/overview |
| SFRA Architecture Guide | https://developer.salesforce.com/docs/commerce/sfra/guide/sfra-overview.html |
| Cartridges | https://developer.salesforce.com/docs/commerce/b2c-commerce/guide/b2c-cartridges.html |
| ISML Templates | https://developer.salesforce.com/docs/commerce/b2c-commerce/guide/b2c-isml.html |
| Script API Reference (Developer Doc) | https://salesforcecommercecloud.github.io/b2c-dev-doc/ |
| Commerce APIs (SCAPI) | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| SFRA GitHub Repository | https://github.com/SalesforceCommerceCloud/storefront-reference-architecture |

---

## 2. PWA Kit / Composable Storefront

### What is Composable Storefront

Composable Storefront is Salesforce's decoupled storefront solution for B2C Commerce Cloud. It separates the frontend (presentation layer) from the commerce backend, connecting them exclusively through APIs. The term "composable" refers to the architectural approach where the storefront is assembled from independent, API-connected components rather than running as a monolithic application on the commerce server.

Unlike SFRA, where templates and logic execute on the B2C Commerce application server, Composable Storefront runs on its own dedicated infrastructure (Managed Runtime) and consumes B2C Commerce services through the Shopper Commerce API (SCAPI). The commerce engine remains the same — catalogs, pricing, promotions, checkout — but the way the storefront accesses and presents that data fundamentally changes.

Salesforce packages this solution as two main components:

- **PWA Kit** — the development framework (React-based) for building the storefront application.
- **Managed Runtime** — the hosting and deployment infrastructure provided by Salesforce to run the storefront at scale.

### What is PWA Kit

PWA Kit is an open-source development framework built on React that provides the tools, libraries, and project structure needed to build a B2C Commerce storefront as a Progressive Web Application. It is organized as a monorepo containing:

- **Retail React App** — a complete reference storefront implementation with pages for product listing, product detail, cart, checkout, account management, and store locator. It serves as both a functional starting point and a demonstration of best practices.
- **commerce-sdk-react** — a library of React hooks for interacting with B2C Commerce APIs (SCAPI). Provides typed, declarative access to products, categories, baskets, orders, promotions, and customer data.
- **pwa-kit-dev** — development tools for local development, building, and deploying bundles.
- **pwa-kit-runtime** — the server-side runtime that handles SSR (Server-Side Rendering) and request routing in production.

The Retail React App uses Chakra UI as its component library, providing 50+ accessible UI components with a theming system based on the Styled System Theme Specification. Styling is fully customizable through theme tokens.

### Architectural Model

Composable Storefront follows a decoupled architecture where the frontend and backend are independent systems connected by APIs:

```
┌─────────────────────────────────────────────────────────┐
│                      Browser                             │
│  (React SPA with client-side navigation)                │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                  Managed Runtime                          │
│  ┌─────────┐  ┌──────────┐  ┌────────────────────┐     │
│  │   CDN   │  │   WAF    │  │  Node.js / Express │     │
│  │ (Edge)  │  │          │  │  (SSR + Routing)   │     │
│  └─────────┘  └──────────┘  └────────────────────┘     │
└──────────────────────────┬──────────────────────────────┘
                           │ SCAPI (REST)
┌──────────────────────────▼──────────────────────────────┐
│              B2C Commerce Cloud                           │
│  (Catalog, Pricing, Promotions, Cart, Checkout, etc.)   │
└─────────────────────────────────────────────────────────┘
```

- **Server-Side Rendering (SSR)** — the initial page load is rendered on the Node.js server within Managed Runtime, delivering a fully formed HTML page for performance and SEO. After hydration, the app transitions to a client-side single-page application (SPA) with instant navigation between pages.
- **API-first** — all commerce data is fetched via SCAPI REST endpoints. There is no direct access to the B2C Commerce script engine or internal platform APIs from the storefront code.
- **Proxy layer** — Managed Runtime includes a proxy server that accelerates API requests between the storefront and B2C Commerce, reducing latency by keeping connections warm and routing efficiently.

### Managed Runtime

Managed Runtime is the hosting infrastructure that Salesforce provides specifically for running Composable Storefront applications. It is not a generic cloud hosting service — it is purpose-built for PWA Kit projects.

Key characteristics:

- **CDN with edge caching** — requests are cached at the edge, close to end users, for fast page delivery.
- **Web Application Firewall (WAF)** — protects environments from security threats automatically.
- **Auto-scaling microservices** — environments scale up and down based on traffic without manual intervention.
- **Multiple environments** — teams can create unlimited environments (development, staging, QA, production) at no additional cost. Environments are lightweight and can be provisioned or removed on demand.
- **Regional deployment** — admins can configure which regions the storefront is deployed to, optimizing latency for target audiences.
- **Bundle-based deployment** — code is packaged into bundles (max 400 MB) and deployed via Runtime Admin or the Managed Runtime API. Initial deployments take up to an hour; subsequent deployments complete in approximately one minute.

### Relationship with B2C Commerce Cloud

The commerce backend remains B2C Commerce Cloud — the same engine that powers SFRA. Merchants still use Business Manager to manage catalogs, pricing, promotions, and campaigns. The difference is how the storefront accesses this data:

| Aspect | SFRA | Composable Storefront |
|--------|------|----------------------|
| Data access | Platform Script API (internal, server-side) | SCAPI (external REST API) |
| Rendering | Server-side on B2C Commerce app server | SSR on Managed Runtime + client-side SPA |
| Hosting | B2C Commerce infrastructure | Managed Runtime (separate) |
| Merchandising | Business Manager → immediate reflection | Business Manager → exposed via SCAPI |

Business Manager functionality is preserved — merchants manage the same commerce data. The storefront simply consumes it differently.

### Authentication Model

Composable Storefront uses **SLAS (Shopper Login and API Access Service)** for authentication. SLAS provides:

- Guest and registered shopper token management
- OAuth 2.0 flows for secure API access
- Session bridging between the storefront and B2C Commerce

This is a separate authentication layer from what SFRA uses internally, designed specifically for API-first access patterns.

### Page Designer

Page Designer integrates with Composable Storefront through a headless rendering model. Component types are built as React components that render within the PWA Kit application. Merchants use the same visual drag-and-drop editor in Business Manager to create and arrange pages.

How it works:

- All Page Designer metadata files (pages, components, aspect types) must include `"arch_type": "headless"` to enable React rendering.
- Components are lazy-loaded on demand through a component registry system, which reduces the initial bundle size — only the components used on a given page are downloaded.
- The visual editor in Business Manager loads the actual PWA Kit storefront in an iframe, so merchants preview exactly what the live page will look like at the real route.
- Developers build component types as React components that receive their configuration as props from Page Designer.

The architecture follows a three-tier hierarchy: Pages (top-level containers with route definitions) → Regions (content areas that hold ordered component lists) → Components (leaf components for content rendering, layout components for organizing nested regions).

Merchants can schedule page publication, preview in context, and manage versions — all from Business Manager without code deployments.

Requirements: PWA Kit version 2.7.0 or later, and a SLAS client configured with the `sfcc.shopper-experience` scope.

### Search and Discovery Integrations

Within the Composable Storefront ecosystem, Salesforce supports integrations with specialized search providers:

- **Algolia** — real-time search, faceting, and merchandising
- **Coveo** — AI-powered search and recommendations
- **Constructor** — product discovery and personalization

These integrations work alongside B2C Commerce's native search capabilities, giving teams the option to enhance product discovery without leaving the Composable Storefront ecosystem.

### Development Workflow

The development experience with PWA Kit is fundamentally different from SFRA:

- **Local development** — developers run the full application locally on their machine with hot module replacement. No remote sandbox uploads needed for frontend iteration.
- **Node.js runtime** — the app runs on a standard Node.js/Express server, both locally and in production.
- **npm ecosystem** — full access to the npm package registry for third-party libraries and tools.
- **TypeScript support** — optional TypeScript for type safety across the codebase.
- **Testing** — Jest and React Testing Library for unit and integration testing, Lighthouse for performance monitoring.
- **Deployment** — bundles are pushed to Managed Runtime via CLI (`pwa-kit-dev push`) and deployed through Runtime Admin or API.

The feedback loop is fast: code locally, see changes instantly, test with real API data via SCAPI, then push and deploy when ready.

### Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js / Express (on Managed Runtime) |
| Framework | React |
| Language | JavaScript / TypeScript |
| UI library | Chakra UI |
| Routing | React Router |
| API client | commerce-sdk-react (SCAPI hooks) |
| SSR | Server-side rendering with hydration |
| Styling | Chakra UI theme system (Styled System) |
| Build tools | Webpack, Babel |
| Testing | Jest, React Testing Library, Lighthouse |
| Deployment | pwa-kit-dev CLI → Managed Runtime |
| Hosting | Managed Runtime (CDN + WAF + auto-scaling) |
| Authentication | SLAS (Shopper Login and API Access Service) |
| APIs consumed | SCAPI (Shopper Commerce API — external REST) |

### Code Examples

#### React Component — Product Detail Page

```jsx
import React from 'react'
import { useProduct } from '@salesforce/commerce-sdk-react'
import { Box, Heading, Text, Button, Stack, Image } from '@chakra-ui/react'

const ProductDetail = ({ productId }) => {
    const { data: product, isLoading } = useProduct({ parameters: { id: productId } })

    if (isLoading) return <Box>Loading...</Box>

    return (
        <Box maxW="container.lg" mx="auto" p={6}>
            <Stack direction={['column', 'row']} spacing={8}>
                <Image
                    src={product?.imageGroups?.[0]?.images?.[0]?.link}
                    alt={product?.name}
                    boxSize="400px"
                    objectFit="cover"
                />
                <Stack spacing={4}>
                    <Heading as="h1" size="lg">
                        {product?.name}
                    </Heading>
                    <Text fontSize="xl" fontWeight="bold">
                        ${product?.price}
                    </Text>
                    <Text>{product?.shortDescription}</Text>
                    <Button colorScheme="blue" size="lg">
                        Add to Cart
                    </Button>
                </Stack>
            </Stack>
        </Box>
    )
}

export default ProductDetail
```

#### Commerce SDK React — Add to Cart Hook

```jsx
import { useShopperBasketsMutation } from '@salesforce/commerce-sdk-react'

const AddToCartButton = ({ productId, quantity = 1 }) => {
    const addItemToBasket = useShopperBasketsMutation('addItemToBasket')

    const handleAddToCart = async () => {
        await addItemToBasket.mutate({
            parameters: { basketId: currentBasketId },
            body: [{ productId, quantity }]
        })
    }

    return (
        <Button
            onClick={handleAddToCart}
            isLoading={addItemToBasket.isLoading}
            colorScheme="blue"
        >
            Add to Cart
        </Button>
    )
}
```

#### Route Configuration

```jsx
import { RouteConfig } from '@salesforce/pwa-kit-runtime/ssr/universal/components/_app-config'

const routes = [
    {
        path: '/',
        component: Home,
        exact: true
    },
    {
        path: '/category/:categoryId',
        component: ProductList
    },
    {
        path: '/product/:productId',
        component: ProductDetail
    },
    {
        path: '/cart',
        component: Cart
    },
    {
        path: '/checkout',
        component: Checkout
    }
]

export default routes
```

### When to Use Composable Storefront

Composable Storefront is the second degree of implementation on B2C Commerce Cloud. It is the path for teams that want full control over the frontend experience using modern web technologies (React, Node.js) while remaining within Salesforce's managed ecosystem for hosting and deployment. The commerce engine stays the same — the difference is how you build and deliver the customer experience.

### Strengths

- Full control over the frontend with React and the modern JavaScript ecosystem.
- Local development with hot reload — no remote sandbox dependency for frontend work.
- Progressive Web App capabilities (offline support, installable, app-like experience).
- Server-side rendering for SEO and performance, with client-side SPA navigation for instant page transitions.
- Managed Runtime eliminates infrastructure concerns — CDN, WAF, scaling, and deployment are handled by Salesforce.
- Access to the npm ecosystem for third-party integrations (Algolia, Coveo, Constructor, analytics, etc.).
- Separation of frontend and backend teams — independent release cycles.

### Considerations

- Requires React expertise — the team needs frontend engineers comfortable with modern JavaScript/TypeScript development.
- The commerce data is accessed exclusively via SCAPI, which may have different capabilities or latency characteristics compared to the internal Platform Script API used by SFRA.
- Two systems to understand: the frontend application (PWA Kit) and the commerce backend (B2C Commerce + Business Manager).
- Bundle size limits (400 MB total) require attention to asset optimization.

### Official Documentation

| Resource | URL |
|----------|-----|
| Composable Storefront Overview | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/overview |
| PWA Kit Architecture | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/architecture.html |
| Getting Started | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/getting-started.html |
| Retail React App | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/retail-react-app.html |
| Deploying Bundles | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/pushing-and-deploying-bundles.html |
| SCAPI Reference | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| commerce-sdk-react | https://developer.salesforce.com/docs/commerce/pwa-kit-managed-runtime/guide/commerce-sdk-react.html |
| PWA Kit GitHub Repository | https://github.com/SalesforceCommerceCloud/pwa-kit |

---

## 3. Headless Commerce

### What is Headless in the Context of B2C Commerce Cloud

Headless commerce is an architectural pattern where B2C Commerce Cloud operates exclusively as a commerce engine — exposing its capabilities (catalog, pricing, promotions, cart, checkout, orders, customer management) through REST APIs — while the presentation layer is built, hosted, and managed entirely outside of Salesforce infrastructure.

In this model, there is no Managed Runtime, no PWA Kit, no Salesforce-hosted frontend. The organization takes full ownership of the experience layer: choosing the frontend framework, the hosting provider, the CDN, the CMS, the search engine, and every other component in the stack. B2C Commerce Cloud becomes one service among many in a composed architecture.

This is where the platform aligns with a **MACH architecture** strategy:

- **Microservices** — each capability (commerce, content, search, payments, personalization) is an independent service with its own lifecycle.
- **API-first** — all communication between services happens through well-defined APIs. B2C Commerce Cloud exposes SCAPI and OCAPI as its integration surface.
- **Cloud-native** — services run on cloud infrastructure designed for elasticity, resilience, and global distribution.
- **Headless** — the frontend is completely decoupled from any backend service, free to consume multiple APIs and render experiences without platform constraints.

Salesforce does not participate in the MACH Alliance, but B2C Commerce Cloud's API layer provides the technical foundation for an organization to build a MACH-compliant architecture using Salesforce as the commerce engine.

### Architectural Model

In a headless architecture, the organization assembles the full stack:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser / App                              │
│  (Any framework: Next.js, Nuxt, Astro, Remix, Angular, Vue,    │
│   Flutter, React Native, native iOS/Android, kiosk, IoT...)     │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                     BFF / API Gateway                             │
│  (Custom orchestration layer — optional but common)              │
└──┬──────────────┬──────────────┬──────────────┬─────────────────┘
   │              │              │              │
   ▼              ▼              ▼              ▼
┌──────┐   ┌──────────┐   ┌─────────┐   ┌──────────────┐
│ CMS  │   │  Search  │   │Commerce │   │  Payments    │
│      │   │          │   │ Engine  │   │              │
│Content│   │Algolia   │   │  B2C    │   │ Stripe      │
│ful   │   │Coveo     │   │Commerce │   │ Adyen       │
│Strapi│   │Construc- │   │ Cloud   │   │ Checkout    │
│Sanity│   │ tor      │   │ (SCAPI) │   │ .com        │
└──────┘   └──────────┘   └─────────┘   └──────────────┘
```

Each service is independently:
- Deployed and scaled
- Versioned and updated
- Monitored and maintained
- Replaced if a better alternative emerges

### B2C Commerce Cloud as a Commerce Engine

In a headless architecture, B2C Commerce Cloud provides the core commerce services through its APIs:

**Shopper APIs (SCAPI)** — consumer-facing, secured via SLAS:
- Shopper Products (catalog browsing, product detail, variations)
- Shopper Search (product search, category navigation, refinements)
- Shopper Baskets (cart management, add/remove items, coupons)
- Shopper Orders (order placement, order history)
- Shopper Customers (registration, login, profiles, addresses, payment instruments)
- Shopper Promotions (active promotions, campaign-driven offers)
- Shopper Gift Certificates (validation and redemption)
- Shopper Login (SLAS — OAuth 2.0 token management)

**OCAPI (Open Commerce API)** — legacy but still operational for scenarios where SCAPI coverage is incomplete:
- Shop API (shopper-facing operations)
- Data API (admin/back-office operations)

**SLAS (Shopper Login and API Access Service)** — authentication layer:
- Public clients (browser, mobile apps — no secret)
- Private clients (BFF, server-side — with secret)
- Guest and registered shopper flows
- OAuth 2.0 with configurable scopes

The organization consumes only the APIs it needs. Business Manager remains the merchandising back-office — catalogs, pricing, promotions, and campaigns are still managed there and exposed through the APIs.

### The Role of the BFF (Backend for Frontend)

In most headless implementations, a BFF (Backend for Frontend) layer sits between the frontend and the various backend services. This is not a Salesforce component — it is built and owned by the organization.

The BFF serves multiple purposes:

- **API orchestration** — combining data from multiple services (commerce + CMS + search + personalization) into a single response optimized for the frontend.
- **Data transformation** — reshaping API responses to match the exact data structure the frontend components expect.
- **Caching** — implementing custom caching strategies per endpoint, reducing API calls and improving response times.
- **Authentication management** — handling SLAS token lifecycle (acquisition, refresh, storage) server-side, keeping secrets out of the browser.
- **Rate limiting and circuit breaking** — protecting against cascading failures when a downstream service degrades.
- **API versioning abstraction** — insulating the frontend from breaking changes in backend API versions.

Common BFF implementations:
- Node.js / Express or Fastify
- Next.js API routes
- Cloudflare Workers / Vercel Edge Functions
- AWS Lambda / API Gateway
- GraphQL federation layer (Apollo Gateway, Mesh)

### Frontend Freedom

The headless model imposes no constraints on the presentation layer. The frontend can be:

| Technology | Use case |
|-----------|----------|
| Next.js (React) | Full-featured web storefront with SSR/SSG/ISR |
| Nuxt (Vue) | Vue-based storefront with server-side rendering |
| Astro | Content-heavy storefronts with minimal JavaScript |
| Remix | Web standards-focused React framework |
| Angular | Enterprise teams with Angular expertise |
| React Native / Flutter | Native mobile commerce apps |
| Swift / Kotlin | Fully native iOS / Android apps |
| Custom SPA | Any JavaScript framework or vanilla JS |
| IoT / Kiosk | Embedded devices, point-of-sale, smart displays |

The choice depends entirely on the organization's team skills, performance requirements, SEO needs, and target platforms.

### Content Management in Headless

In this model, Page Designer (Business Manager) is typically not used for content management. Instead, the organization selects a headless CMS that serves content via API:

- **Contentful** — structured content with a rich API and CDN delivery
- **Sanity** — real-time collaborative editing with GROQ query language
- **Strapi** — open-source, self-hosted headless CMS
- **Amplience** — commerce-focused content management with scheduling
- **Contentstack** — enterprise headless CMS with workflow automation

The CMS manages editorial content (banners, landing pages, blog posts, promotional content), while B2C Commerce manages commerce content (products, categories, pricing, promotions). The BFF or frontend orchestrates both into a unified experience.

### Search and Discovery

Similarly, product search and discovery can be delegated to specialized services:

- **Algolia** — real-time search with typo tolerance, faceting, and AI recommendations
- **Coveo** — AI-driven search and relevance with machine learning ranking
- **Constructor** — product discovery with personalization and merchandising rules
- **Elasticsearch / OpenSearch** — self-managed search infrastructure

These services index product data from B2C Commerce (via API or data feeds) and provide search results directly to the frontend, bypassing B2C Commerce's native search entirely. This allows for sub-50ms search responses and advanced features like visual search, natural language queries, and AI-powered recommendations.

### Infrastructure Ownership

In a headless architecture, the organization owns and operates:

| Concern | Responsibility |
|---------|---------------|
| Frontend hosting | Vercel, Netlify, AWS CloudFront, Azure CDN, Cloudflare Pages, self-managed |
| CDN / Edge caching | Configured per provider — cache invalidation is your responsibility |
| WAF / Security | Cloud provider WAF, Cloudflare, Akamai, or custom rules |
| Scaling | Auto-scaling policies, serverless functions, container orchestration |
| CI/CD | GitHub Actions, GitLab CI, Jenkins, CircleCI — custom pipelines |
| Monitoring | Datadog, New Relic, Grafana, custom observability stack |
| Error tracking | Sentry, Bugsnag, Rollbar |
| Performance | Core Web Vitals monitoring, synthetic checks, RUM |

This is a significant operational shift compared to SFRA or Composable Storefront, where Salesforce manages hosting, CDN, WAF, and scaling.

### Development Workflow

- **Full local development** — the entire frontend runs locally with complete control over the dev server, build pipeline, and tooling.
- **Framework-native tooling** — use whatever the chosen framework provides (Next.js dev server, Vite, Turbopack, etc.).
- **API mocking** — teams can develop against mock APIs or sandbox instances of B2C Commerce, decoupling frontend velocity from backend availability.
- **Independent deployments** — frontend releases are fully independent of B2C Commerce deployments. Ship frontend changes without touching the commerce backend.
- **Multi-environment strategies** — development, staging, preview, production environments managed through the organization's own infrastructure.
- **Testing at every level** — unit, integration, e2e (Playwright, Cypress), visual regression, performance (Lighthouse CI), accessibility (axe).

### Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | Any (Node.js, Deno, edge functions, serverless, containers) |
| Framework | Any (Next.js, Nuxt, Astro, Remix, Angular, custom) |
| Language | Any (TypeScript, JavaScript, Go, Rust for edge, etc.) |
| UI library | Any (Tailwind, Material UI, custom design system) |
| CMS | Headless CMS (Contentful, Sanity, Strapi, Amplience) |
| Search | Specialized (Algolia, Coveo, Constructor, OpenSearch) |
| Commerce API | SCAPI + OCAPI (B2C Commerce Cloud) |
| Authentication | SLAS (public or private client) |
| BFF | Custom (Node.js, edge functions, GraphQL gateway) |
| Hosting | Any cloud provider (Vercel, AWS, Azure, GCP, Cloudflare) |
| CDN | Provider-managed or custom (CloudFront, Fastly, Akamai) |
| CI/CD | Custom pipelines (GitHub Actions, GitLab CI, etc.) |
| Monitoring | Custom observability stack |

### Code Examples

#### Next.js — Product Detail Page with ISR

```typescript
import { GetStaticProps, GetStaticPaths } from 'next'

interface Product {
    id: string
    name: string
    price: number
    description: string
    imageUrl: string
}

export const getStaticPaths: GetStaticPaths = async () => {
    const res = await fetch(
        `${process.env.SCAPI_BASE_URL}/product/shopper-products/v1/organizations/${process.env.ORG_ID}/products?siteId=${process.env.SITE_ID}&ids=top-seller-1,top-seller-2`,
        { headers: { Authorization: `Bearer ${await getAccessToken()}` } }
    )
    const data = await res.json()

    return {
        paths: data.data.map((p: Product) => ({ params: { pid: p.id } })),
        fallback: 'blocking'
    }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
    const token = await getAccessToken()
    const res = await fetch(
        `${process.env.SCAPI_BASE_URL}/product/shopper-products/v1/organizations/${process.env.ORG_ID}/products/${params?.pid}?siteId=${process.env.SITE_ID}`,
        { headers: { Authorization: `Bearer ${token}` } }
    )
    const product = await res.json()

    return { props: { product }, revalidate: 60 }
}

export default function ProductPage({ product }: { product: Product }) {
    return (
        <div className="max-w-6xl mx-auto p-6 grid grid-cols-1 md:grid-cols-2 gap-8">
            <img src={product.imageUrl} alt={product.name} className="w-full rounded-lg" />
            <div>
                <h1 className="text-3xl font-bold">{product.name}</h1>
                <p className="text-2xl mt-4">${product.price}</p>
                <p className="mt-4 text-gray-600">{product.description}</p>
                <button className="mt-6 bg-blue-600 text-white px-8 py-3 rounded-lg">
                    Add to Cart
                </button>
            </div>
        </div>
    )
}
```

#### BFF — SLAS Token Management (Node.js)

```typescript
import { Router } from 'express'

const router = Router()

const SLAS_BASE = process.env.SLAS_BASE_URL
const CLIENT_ID = process.env.SLAS_CLIENT_ID
const CLIENT_SECRET = process.env.SLAS_CLIENT_SECRET
const ORG_ID = process.env.COMMERCE_ORG_ID
const SITE_ID = process.env.COMMERCE_SITE_ID

router.post('/auth/guest-token', async (req, res) => {
    const response = await fetch(
        `${SLAS_BASE}/shopper/auth/v1/organizations/${ORG_ID}/oauth2/token`,
        {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({
                grant_type: 'client_credentials',
                client_id: CLIENT_ID,
                client_secret: CLIENT_SECRET,
                channel_id: SITE_ID
            })
        }
    )

    const token = await response.json()
    res.json({ access_token: token.access_token, expires_in: token.expires_in })
})

router.post('/auth/login', async (req, res) => {
    const { username, password } = req.body

    const response = await fetch(
        `${SLAS_BASE}/shopper/auth/v1/organizations/${ORG_ID}/oauth2/token`,
        {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({
                grant_type: 'urn:demandware:params:oauth:grant-type:credentials',
                client_id: CLIENT_ID,
                client_secret: CLIENT_SECRET,
                channel_id: SITE_ID,
                username,
                password
            })
        }
    )

    const token = await response.json()
    res.json({ access_token: token.access_token, refresh_token: token.refresh_token })
})

export default router
```

#### API Orchestration — Combining Commerce + CMS

```typescript
import { getCommerceProduct } from './services/commerce'
import { getCmsContent } from './services/cms'
import { getSearchRecommendations } from './services/search'

async function getProductPageData(productId: string) {
    const [product, editorial, recommendations] = await Promise.all([
        getCommerceProduct(productId),
        getCmsContent(`product-${productId}-editorial`),
        getSearchRecommendations(productId)
    ])

    return {
        product: {
            id: product.id,
            name: product.name,
            price: product.price,
            images: product.imageGroups[0].images,
            variants: product.variants,
            inventory: product.inventory
        },
        editorial: {
            headline: editorial.fields.headline,
            body: editorial.fields.richText,
            media: editorial.fields.heroImage.url
        },
        recommendations: recommendations.hits.map(hit => ({
            id: hit.productId,
            name: hit.productName,
            price: hit.price,
            image: hit.thumbUrl
        }))
    }
}
```

### When to Use Headless

Headless is the third degree of implementation on B2C Commerce Cloud. It is the path for organizations that require complete architectural freedom and are prepared to own the full technology stack beyond the commerce engine. It enables a MACH architecture strategy where B2C Commerce Cloud serves as the commerce microservice within a broader composed ecosystem.

### Strengths

- Complete technology freedom — no constraints on framework, language, hosting, or infrastructure decisions.
- True MACH architecture — each component is independently deployable, scalable, and replaceable.
- Multi-channel by design — the same commerce APIs serve web, mobile apps, IoT, kiosks, voice assistants, and any future touchpoint.
- Best-of-breed composition — select the optimal service for each capability (search, CMS, personalization, payments) rather than accepting a single vendor's solution for everything.
- Independent scaling — frontend, BFF, and commerce engine scale independently based on their own traffic patterns.
- Maximum performance control — edge rendering, static generation, incremental regeneration, and custom caching strategies are all available.
- Vendor flexibility — the frontend is not locked to Salesforce infrastructure. If the commerce engine needs to change, only the API integration layer is affected.

### Considerations

- Full infrastructure responsibility — the organization must provision, secure, monitor, and maintain all non-commerce components.
- Higher operational complexity — more moving parts means more potential failure points, more monitoring, more on-call burden.
- Requires strong engineering team — this model demands expertise in cloud infrastructure, API design, frontend architecture, DevOps, and distributed systems.
- Longer time-to-market for initial launch — assembling the stack from scratch takes more upfront investment than starting with SFRA or Composable Storefront.
- No Page Designer — content management requires a separate headless CMS and custom editorial workflows.
- API rate limits and quotas — B2C Commerce Cloud APIs have rate limits that must be factored into caching and orchestration strategies.
- Integration maintenance — when B2C Commerce Cloud releases API updates or deprecations, the organization is responsible for adapting.

### Official Documentation

| Resource | URL |
|----------|-----|
| B2C Commerce API (SCAPI) | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| SLAS Authorization | https://developer.salesforce.com/docs/commerce/commerce-api/guide/authorization-for-shopper-apis.html |
| OCAPI Reference | https://developer.salesforce.com/docs/commerce/b2c-commerce/references/b2c-commerce-ocapi/ |
| B2C Commerce Developer Resources | https://salesforcecommercecloud.github.io/b2c-dev-doc/ |
| commerce-sdk (TypeScript) | https://github.com/SalesforceCommerceCloud/commerce-sdk |
| commerce-sdk-isomorphic | https://github.com/SalesforceCommerceCloud/commerce-sdk-isomorphic |
