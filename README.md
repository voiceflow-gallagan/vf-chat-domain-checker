
# Voiceflow Chat Widget Domain Checkeer

## Cloudflare Worker

Update the following code accordingly with your project ID and the allowed domains.
Create a new worker, paste the code, and deploy it.

Use the deployement URL in the Voiceflow Chat Widget code snippet in next step.

```javascript
// Array of allowed domains
const allowedDomains = ['voiceflow-gallagan.github.io','dash.cloudflare.com', 'google.com'];

// Array of allowed domain patterns
const allowedDomainPatterns = [/^.*\.voiceflow\.com$/];

// The project ID to return if the domain is allowed
const projectID = '66a0e19b4422a15a3960409c';

// A secret key for signing tokens (replace with a strong, unique secret)
const SECRET_KEY = 'your-secret-key-here';


addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

function isAllowedDomain(hostname) {
  // Check exact matches
  if (allowedDomains.includes(hostname)) {
    return true;
  }
  // Check pattern matches
  return allowedDomainPatterns.some(pattern => pattern.test(hostname));
}

async function handleRequest(request) {
  const origin = request.headers.get('Origin') || '*';

  // Handle preflight requests
  if (request.method === 'OPTIONS') {
    return handleCORS(request, origin);
  }

  // Extract the hostname from the origin
  let hostname;
  try {
    hostname = new URL(origin).hostname;
  } catch (error) {
    hostname = origin; // Fallback to the raw origin if it's not a valid URL
  }

  // Check if the hostname is in the allowed domains list
  if (isAllowedDomain(hostname)) {
    // If allowed, generate and return a token
    const token = await generateToken(hostname);
    const response = JSON.stringify({ token, projectID });
    return handleCORS(request, origin, response);
  } else {
    // If not allowed, return an error
    return handleCORS(request, origin, `Forbidden: Domain not allowed`, 403);
  }
}

async function generateToken(hostname) {
  const expirationTime = Date.now() + 1 * 60 * 1000; // Token expires in 1 minute
  const payload = {
    exp: expirationTime,
    hostname: hostname
  };
  const signature = await generateSignature(payload);
  return btoa(JSON.stringify(payload)) + '.' + signature;
}

async function generateSignature(payload) {
  const encoder = new TextEncoder();
  const data = encoder.encode(JSON.stringify(payload) + SECRET_KEY);
  const hash = await crypto.subtle.digest('SHA-256', data);
  return btoa(String.fromCharCode.apply(null, new Uint8Array(hash)));
}

function handleCORS(request, origin, content = null, status = 200) {
  const corsHeaders = {
    'Access-Control-Allow-Origin': origin,
    'Access-Control-Allow-Methods': 'GET, HEAD, POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type',
    'Access-Control-Max-Age': '86400',
  };

  if (request.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders });
  }

  const response = new Response(content, { status });
  Object.keys(corsHeaders).forEach(key => response.headers.set(key, corsHeaders[key]));
  return response;
}
```


## Voiceflow Chat Widget

On you webpage, use the following code snippet to check the domain and load the Chat Widget.
Be sure to replace `https://chatwidget-domain-checker.voiceflow.workers.dev` with the URL of your Cloudflare worker.

```html
<script>
(function(d, t) {
      var v = d.createElement(t), s = d.getElementsByTagName(t)[0];
      v.onload = function() {
        async function getTokenAndProjectID() {
          try {
            const response = await fetch('https://chatwidget-domain-checker.voiceflow.workers.dev/', {
              method: 'GET',
              headers: {
                'Content-Type': 'application/json',
              },
            });
            if (!response.ok) {
              throw new Error(`HTTP error! status: ${response.status}`);
            }
            const data = await response.json();
            return data;
          } catch (error) {
            console.error('Error fetching token and project ID:', error);
            throw error;
          }
        }

        getTokenAndProjectID().then(({ token, projectID }) => {
          console.log('Loading chat widget');
          window.voiceflow.chat.load({
            verify: { projectID: projectID },
            url: 'https://general-runtime.voiceflow.com',
            versionID: 'development',
            launch: {
              event: {
                type: "launch",
                payload: {
                  validationToken: token
                }
              }
            }
          });
        }).catch(error => {
          console.error('Error loading chat widget:', error);
        });
      }
      v.src = "https://cdn.voiceflow.com/widget/bundle.mjs"; v.type = "text/javascript"; s.parentNode.insertBefore(v, s);
    })(document, 'script');
</script>
```

## Voiceflow Agent
Import the `Domain Checker 2024-07-25 11-05.vf` file into your Voiceflow workspace.
You will have to update the code in the **Check Authorization** step to match the domains you want to allow as well as the secret key you set in the Cloudflare worker.
