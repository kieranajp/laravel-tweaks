# The asset pipeline

Laravel Mix is a right bastard. While I'm sure it's simpler than configuring Webpack, it still seems overly complicated to me, especially compared to something like Rollup or Parcel.js. It's possible to get this ease of configuration, but avoid blurring responsibility between build and run time.

## Problem

Often, you'll find lines like this in Blade templates:

```html
<link rel="stylesheet" href="{{ mix('css/app.css') }}" />
```

As far as I can tell, the key benefit of this is that it adds a cachebuster to the URL. However, it requires a generated file, `mix-manifest.json` to be present and correct - if that's not the case it often leads to errors like this:

```
The Mix manifest does not exist.
```

or this: 

```
Unable to locate Mix file: /css/app.css.
```

This can be a real problem, especially when using multi-stage Docker builds. Causing a 500 error due to a failing in the JS compile step is a horrible example of separation of concerns.

## Solution

Rather than spending ages tackling this in vain, we can bypass Mix altogether and add our own simple cachebuster.

I used Helm to add an environment variable `GIT_SHA` to the application charts at deploy time:

```yaml
# charts/foo/configmap.yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: {{ include "chart.fullname" . }}
  labels:
    {{- include "chart.labels" . | nindent 4 }}
data:
  GIT_SHA: "{{ .Values.version }}"
  {{- with .Values.env }}
  {{- toYaml . | nindent 2}}
  {{- end }}

```

Alternatively you could write a version.txt file containing the SHA. Either way it can be read into application config:

```php
// config/app.php

return [
    'name' => env('APP_NAME', 'Laravel'),

    // If you want to read from environment variable (preferred)
    'version' => env('GIT_SHA', 'dev'), 

    // OR: If you can't dynamically set env variables at release
    'version' => @file_get_contents('../storage/app/version.txxt') ?: 'dev'

    // ...
];
```

You can then remove this Mix function from server-side code entirely:

```html
<link rel="stylesheet" href="/css/app.css?v={{ config('app.version') }}" />
```

Enjoy some semblance of separation of concerns on your way to the madhouse.