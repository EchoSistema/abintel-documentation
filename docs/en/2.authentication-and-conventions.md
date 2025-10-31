## 2. Authentication & Conventions

### Required Headers

| **Header**          | **Purpose**         | **Notes**                                                                 |
| ------------------- | ------------------- | ------------------------------------------------------------------------- |
| `X-Customer-Api-Id` | Tenant UUID (v4)    | ✅ **Mandatory** if no admin key is used                                   |
| `X-Secret`          | 64-character secret | 🔁 Rotate every **90 days**                                               |
| `Accept-Language`   | Response language   | 🌐 Options: `en` (default), `es`, `pt` <br>🔄 Overrides body `lang` param |