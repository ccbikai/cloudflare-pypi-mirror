
# Cloudflare PyPI Mirror

简体中文 | [English](./README.en.md)

---

Python 的包管理器是 pip, 使用的包索引服务器是 PyPI 。

PyPI 在中国大陆是无法正常访问的，但是有许多的 Mirror。清华、阿里云、腾讯云、华为云等不少网站都提供了镜像。这些镜像除了清华的 tuna，其他都不支持 JSON-based Simple API for Python ([PEP 691](https://peps.python.org/pep-0691/))。

[Pyodide](https://micropip.pyodide.org/en/stable/index.html) 是一个在 WebAssembly 中运行 Python 的工具库，使用 [Micropip](https://micropip.pyodide.org/en/stable/index.html) 通过 PyPI 来安装包。由于 WebAssembly 在浏览器内运行需要跨域和 PEP 691，但是清华的 tuna 又不支持 CORS 跨域。

所以在中国大陆可能没有 Micropip 可用的 PyPI 镜像。 基于这个背景，我们使用 Cloudflare 搭建一个支持 PEP691 和 CORS 的 Mirror。

## 部署方式

### [Workers](https://workers.cloudflare.com/)

使用下面的代码创建一个 Worker, 绑定一个自定义路由。

优点：免费计划可用。
缺点：会产生很多 Worker 请求，可能超出免费计划后不可用或需要付费。

### [Snippets](https://developers.cloudflare.com/rules/snippets/)

使用下面的代码创建一个 Snippet, 并将对应域名的流量分配给 Snippet 处理。

优点：不产生 Worker 请求，支持大量使用。
缺点：Snippets 目前只有 Pro 以上计划使用，Free 不可用。

## 代码

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

      // 缓存头部未支持跨域，导致页面跨域错误
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
