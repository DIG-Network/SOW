# Statement of Work
## DIG Network URN-Aware Reverse Proxy Implementation

### Project Overview
Development of a NodeJS-based reverse proxy service that acts as a middleware layer for the DIG Network, routing traffic to locally-hosted Docker services. The proxy will handle URN-encoded subdomains and manage access to two separate backend servers:
1. Content Server - Handles primary content delivery and URN resolution
2. Propagation Server - Manages network propagation and relay functions

Both backend servers will be deployed as Docker containers on the same host machine as the proxy. The proxy serves as the entry point for all external traffic, directing requests to the appropriate local service based on subdomain patterns.

### Objectives
- Create a reverse proxy for a single configured hostname
- Implement URN subdomain detection and decoding using dig-sdk
- Enable automatic SSL certificate provisioning via Let's Encrypt
- Provide seamless proxying to content servers based on decoded URNs

### Technical Requirements

#### 2. Access Patterns

The proxy supports two operational configurations:

1. Hostname + IP mode (when HOSTNAME is configured):
   - Both access patterns are supported simultaneously:
     * Full hostname-based access with subdomains:
       - `node.{hostname}` - Direct proxy to content server
       - `relay.{hostname}` - Direct proxy to propagation server
       - `*.{hostname}` - Dynamic URN-encoded content resolution (adds header "x-spa": true)
       - Default content access: `node.{hostname}/urn`
     * Full IP-based direct access:
       - Content server: `https://{ip}:{port}/urn:dig:chia:....`
       - Propagation server: `https://{ip}:{port}/relay/...`

2. IP-only mode (default when no HOSTNAME configured):
   - Content server: `https://{ip}:{port}/urn:dig:chia:....`
   - Propagation server: `https://{ip}:{port}/relay/...`
   - Support both IPv4 and IPv6 addresses
   - Reserved path '/relay' for propagation server access
   - All other paths assumed to be URN access for content server

Both modes support mTLS passthrough:
- Forward client certificates from incoming requests to backend servers
- Forward server certificates from backend responses
- No direct mTLS handling by the proxy itself
- Backend servers (content and propagation) handle all mTLS authentication

Special Header Requirements:
- When using encoded subdomains (*.{hostname}), the proxy must add "x-spa": true header to the request
- This header should only be added for encoded subdomain access
- Header should not be added for direct node/relay subdomain access or IP-based access

#### 2. URN Subdomain Handling
- Integrate with dig-sdk for URN encoding/decoding
- Support two access patterns for URN resources:
  1. Direct URN access via `node.{hostname}/urn`
  2. Encoded subdomain access via `{encoded-urn}.{hostname}`
- Handle encoded subdomain resolution by proxying to appropriate content server URN path
- Example: `3idfhfsekjee.example.com` -> `node.example.com/urn:dig:chia:storid/key`
- Validate decoded URNs against DIG Network specifications
- Handle invalid or malformed URN gracefully with appropriate error responses
- Support the URN format: urn:dig:chia:{storeID}:{optional roothash}/{optional resource key}

#### 3. Request Routing
- Detect access pattern (hostname vs IP) based on incoming request
- For hostname access:
  * Route based on subdomain patterns
  * Handle URN encoding/decoding for wildcard subdomains
  * Add "x-spa": true header for encoded subdomain requests
- For IP-based access:
  * Route based on path prefix
  * Reserve '/relay' path for propagation server
  * All other paths route to content server
  * Validate URN format for content server requests
- Implement request forwarding to appropriate local servers:
  * Content Server container (default: localhost:4161)
  * Propagation Server container (default: localhost:4159)
- Pass through mTLS certificates:
  * Forward client certificates in requests to backend
  * Forward server certificates in responses from backend
- Monitor health of both backend Docker containers
- Handle all HTTP methods
- Support WebSocket connections and proper upgrade handling
- Maintain original request headers and parameters
- Implement comprehensive request/response logging
- Handle timeout configurations
- Implement retry logic for failed requests
- Support graceful handling of container restarts

#### 4. SSL Certificate Management
When HOSTNAME is configured:
- Check for valid SSL certificates on startup
- If valid certificates are not detected:
  1. Generate preliminary SSL setup requirements
  2. Serve a dedicated SSL setup instruction page for all incoming requests
  3. Include DNS challenge records that need to be added for wildcard domain validation
  4. Provide clear step-by-step instructions for completing SSL setup
  5. Include verification steps to confirm DNS propagation
  6. Offer automatic certificate generation once DNS challenges are verified
- Support wildcard certificate generation for *.{hostname}
- Certificate renewal management
- Clear console output for certificate status
- Proper storage and management of certificate files

When no HOSTNAME is configured:
- Skip Let's Encrypt certificate provisioning
- Focus only on managing mTLS certificates for backend communication

##### SSL Setup Instruction Page Requirements
- Clean, user-friendly interface
- Copy-paste friendly DNS record format
- Clear visual indication of setup progress
- Automatic DNS record verification checking
- Refresh instructions for certificate generation once DNS is verified
- Troubleshooting guidance for common issues
- Contact/support information for additional help

#### Error Handling
- Redirect all non-matching requests to a branded 404 page including:
  - Requests to undefined subdomains
  - Malformed URN encodings in subdomains
  - Invalid path requests
  - Non-existent resources
- Implement custom 404 page with:
  - DIG Network branding
  - Clear error explanation
  - Consistent styling across all error scenarios
  - Optional helpful links or guidance
- Maintain proper HTTP status codes while serving branded error page
- Log all redirects to error page for monitoring
- Include request details in logs for debugging

### Environment Variables
```
# Required
NODE_ENV=           # Environment (development/production)

# Optional with defaults
HOSTNAME=           # Hostname for subdomain-based access
PORT=               # Proxy listen port (default: 80)
CONTENT_SERVER=     # Content server URL (default: http://localhost:4161)
RELAY_SERVER=       # Relay server URL (default: http://localhost:4159)
CERT_EMAIL=         # Let's Encrypt email (required if HOSTNAME set)

# Rate Limiting
RATE_LIMIT=         # Requests per minute per IP (default: 60)
RATE_BURST=         # Concurrent requests per IP (default: 10)
RATE_WINDOW=        # Time window in seconds (default: 60)
RATE_BAN_TIME=      # Ban duration in seconds (default: 3600)
```

Service Defaults:
- Reverse Proxy: localhost:80
- Content Server: localhost:4161
- Relay Server: localhost:4159

The proxy can operate in two modes:
1. Full hostname mode (when HOSTNAME is set): supports both subdomain-based and IP-based access
2. IP-only mode (when HOSTNAME is not set): supports only IP-based direct access

Service Defaults:
- Reverse Proxy: localhost:80
- Content Server: localhost:4161
- Relay Server: localhost:4159

If not set, the environment variables will default to their respective localhost URLs and ports. All servers should be running as Docker containers on the local machine.

### Docker Integration
- The proxy service should be designed to work alongside other DIG Network Docker containers
- Assumes content and propagation servers are running in separate containers on the same host
- Should handle container networking within the local Docker environment
- Must support container restarts and temporary outages
- Should integrate with Docker logs for unified logging

### Testing Requirements
The codebase must maintain 100% test coverage across all functionality including:
- Core proxy logic
- URN handling
- SSL certificate management
- Error scenarios
- Configuration management
- All utility functions

Coverage reports must be generated as part of CI/CD and no code shall be merged that reduces coverage below 100%.

### Technical Constraints
- Must be implemented in NodeJS
- Must use dig-sdk for URN handling
- Must support Node.js LTS versions
- Must follow security best practices for proxy servers
- Must implement proper rate limiting and security measures

### Security Requirements
- Input validation for all URN components
- Protection against common proxy vulnerabilities
- Secure certificate handling
- Proper error message sanitization
- Rate limiting implementation
- DOS protection measures