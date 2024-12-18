# Cloudflare PyPI Mirror

[简体中文](./README.md) | English

---

Python's package manager pip uses PyPI (Python Package Index) as its default package repository.

PyPI is not reliably accessible from mainland China, but there are several mirrors available. While institutions like Tsinghua University, Alibaba Cloud, Tencent Cloud, and Huawei Cloud provide mirrors, only Tsinghua's TUNA mirror supports the JSON-based Simple API for Python ([PEP 691](https://peps.python.org/pep-0691/)).

[Pyodide](https://micropip.pyodide.org/en/stable/index.html) is a library that runs Python in WebAssembly and uses [Micropip](https://micropip.pyodide.org/en/stable/index.html) to install packages from PyPI. Since WebAssembly running in browsers requires CORS and PEP 691 support, and Tsinghua's TUNA mirror doesn't support CORS, there are effectively no PyPI mirrors available in mainland China that work with Micropip.

Given this situation, we've created a Cloudflare-based mirror that supports both PEP 691 and CORS.

## Deployment Options

### [Workers](https://workers.cloudflare.com/)

Create a Worker with the code below and bind it to a custom route.

Pros: Available on the free plan.

Cons: Generates many Worker requests, which may exceed free tier limits and require paid plans.

### [Snippets](https://developers.cloudflare.com/rules/snippets/)

Create a Snippet with the code below and route domain traffic through it.

Pros: Doesn't generate Worker requests, suitable for high-volume usage.

Cons: Currently only available on Pro plans and above, not available on Free tier.

## Code

```js
const PyPI = 'https://pypi.org'
const PyPI_FILES = 'https://files.pythonhosted.org'

export default {
  async fetch(request) {
    const url = new URL(request.url)

    if (url.pathname.startsWith('/pypi/')) {
      const pypiPath = url.pathname.replace('/pypi', '')
      const pypiUrl = `${PyPI}${pypiPath}`
      const response = await fetch(pypiUrl, request)
      let body = await response.text()

      body = body.replace(new RegExp(PyPI_FILES, 'g'), `${url.origin}/files`)

      // Cache headers don't support CORS, which causes cross-origin errors
      const headers = new Headers(response.headers)
      headers.delete('ETag')

      return new Response(body, {
        headers,
      })
    }

    if (url.pathname.startsWith('/files/')) {
      const filePath = url.pathname.replace('/files', '')
      const fileUrl = `${PyPI_FILES}${filePath}`
      return fetch(fileUrl, request)
    }

    return new Response('PyPI Works', { status: 200 })
  },
}
```
