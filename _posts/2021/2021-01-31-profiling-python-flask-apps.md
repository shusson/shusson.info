# Profiling python flask apps

**31/01/2021**

Python provides a couple standard [profilers](https://docs.python.org/3/library/profile.html). They can be used to determine how often and for how long various parts of the program executed, which is particularly useful if you are debugging a flask app. The Werkzeug library provides a middleware that profiles each request with the cProfile module.

Example:

```python
app.wsgi_app = ProfilerMiddleware(app.wsgi_app, profile_dir="profiles")
```

Which will produce a `xxx.prof` file per request in the `profiles` directory.

There are a few tools around to visualize the profiles. e.g [snakeviz](https://jiffyclub.github.io/snakeviz).

```bash
pip install snakeviz
snakeviz profiles/POST.x.y.z.1000ms.1611787172.prof
```
