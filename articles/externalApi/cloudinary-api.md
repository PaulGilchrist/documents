# Cloudinary API Integration

This document will summarize the steps for using cloudinary for retrieving and displaying Cloudinary sized and cached images within an Angular application.

## Steps

1. Add the following two libraries to `package.json` dependencies.

```js
"@cloudinary/angular-5.x": "^1.1.0",
"cloudinary-core": "^2.8.0"
```

1. Add `cloud_name` property to the environment files.  Since this is a read-only integration, and Cloudinary only has production data, each environment will use the same `cloud_name`.

2. Import the Cloudinary module into Angular's `app.module`

```js
import { CloudinaryModule } from '@cloudinary/angular-5.x';
import * as  Cloudinary from 'cloudinary-core';
imports: [
    CloudinaryModule.forRoot(Cloudinary, { cloud_name: environment.cloud_name})
]
```

3. Add Cloudinary image components into your HTML as in the following example:

```html
<cl-image
    class="rounded w-100 h-100"
    public-id="{{ plan.Images[0].Path }}"
    format="auto"
    quality="auto"
    width="auto"
    type="fetch"
    crop="fill"
    [attr.alt]="plan.Images[0].AltText ? plan.Images[0].AltText : ''"
    [attr.aspect-ratio]="1.5">
</cl-image>
```

More Angular specific details are available on Cloudinary's website at [https://cloudinary.com/documentation/angular_integration](https://cloudinary.com/documentation/angular_integration)