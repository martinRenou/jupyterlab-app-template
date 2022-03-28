# Build your own Lab remix

This documentation will guide you through making your own web application using the toolset provided by the JupyterLab plugin system and JupyterLab UI components.

It is advised to also look at existing Lab remixes implementations and examples.

- Retrolab is a custom Lab remix which gives a class Jupyter Notebook look to JupyterLab: https://github.com/jupyterlab/retrolab
- Quetz-frontend is the website implementation for the open source package server Quetz. The website is completely unrelated to Jupyter, but was built using the JupyterLab remix approach:  https://github.com/mamba-org/quetz-frontend
- You can find multiple examples of custom web applications using JupyterLab components under [the examples directory of the JupyterLab repo](https://github.com/jupyterlab/jupyterlab/tree/master/examples). These are good starting points for making simple apps.
- You can also look at the [jupyterlab-app-template repository](https://github.com/jtpio/jupyterlab-app-template) which provides a template for starting a more advanced application, with everything setup for packaging your application. This template was used for making the Quetz frontend.

## Structure of a Lab remix

Like any other web application, one built using JupyterLab components requires you implement both a backend and a frontend. The backend serves all the frontend assets to the user (HTML, CSS and JavaScript code, images, data etc) by implementing the HTTP endpoints. The frontend is made of the JavaScript code that runs in the user's browser to make dynamic pages.

### Server - backend

In order to implement the server application (in Python), you just need to subclass the `jupyterlab_server.LabServerApp` class. This class will provide everything for starting the web app from Python, this is where you will setup the right handlers for the different HTTP endpoints.

In the following example, we create a custom Lab application with a single handler for the `http://localhost:8888/custom` endpoint, which is the main entry for our application.

Let's put the following content in our `main.py` file:

```python
import os

import tornado

from jupyter_server.utils import url_path_join as ujoin
from jupyter_server.base.handlers import JupyterHandler
from jupyter_server.extension.handler import (
    ExtensionHandlerMixin,
    ExtensionHandlerJinjaMixin
)

from jupyterlab_server import LabServerApp

HERE = os.path.dirname(__file__)

version = '0.1.0'

def _jupyter_server_extension_points():
    return [
        {
            'module': __name__,
            'app': CustomApp
        }
    ]


class CustomHandler(
    ExtensionHandlerJinjaMixin,
    ExtensionHandlerMixin,
    JupyterHandler
    ):
    """Main handler for our application."""

    @tornado.web.authenticated
    def get(self):
        """Implement the HTTP GET request.
        In this case get the main page for the application's interface.
        """
        config_data = {
            # Use camelCase here, since that's what the lab components expect
            "appVersion": version,
            'baseUrl': self.base_url,
            'token': self.settings['token'],
            'fullStaticUrl': ujoin(self.base_url, 'static', self.name),
            'frontendUrl': ujoin(self.base_url, 'custom/'),
        }
        # Render the main template with Jinja
        return self.write(
            self.render_template(
                'index.html',
                static=self.static_url,
                base_url=self.base_url,
                token=self.settings['token'],
                page_config=config_data
                )
            )


class CustomApp(LabServerApp):
    """You custom Lab remix application."""

    name = __name__
    app_name = 'JupyterLab Custom App'
    app_version = version

    # Main endpoint to your application (where index.html will be served)
    extension_url = '/custom'

    # This is the default URL that the user is redirected
    # to when arriving on the web app
    default_url = '/custom'

    # Whether or not to load the other JupyterLab extensions
    load_other_extensions = False

    # Directory that contains the static assets (index.html, CSS, images etc)
    static_dir = os.path.join(HERE, 'static')
    app_settings_dir = os.path.join(HERE, 'static', 'application_settings')
    schemas_dir = os.path.join(HERE, 'static', 'schemas')
    templates_dir = os.path.join(HERE, 'templates')
    themes_dir = os.path.join(HERE, 'static', 'themes')
    user_settings_dir = os.path.join(HERE, 'static', 'user_settings')
    workspaces_dir = os.path.join(HERE, 'static', 'workspaces')

    def initialize_handlers(self):
        """Add custom handler to Lab Server's handler list.

        Handlers is a list of tuples (endpoint, handler_cls)
        """
        self.handlers.append(
            ('/custom', CustomHandler)
        )

if __name__ == '__main__':
    # Launch the server application
    CustomApp.launch_instance()
```

You can now start your lab extension running `python main.py` from your terminal (this will not work until we have a proper frontend to serve):

```
[I 2022-03-28 10:17:38.008 ServerApp] __main__ | extension was successfully linked.
[I 2022-03-28 10:17:38.032 ServerApp] __main__ | extension was successfully loaded.
[I 2022-03-28 10:17:38.032 ServerApp] __main__ is running without loading other extensions.
[I 2022-03-28 10:17:38.033 ServerApp] Serving notebooks from local directory: /home/user/custom
[I 2022-03-28 10:17:38.033 ServerApp] Jupyter Server 1.13.5 is running at:
[I 2022-03-28 10:17:38.033 ServerApp] http://localhost:8888/custom?token=a59dee2ea49885c518515071ab058de33e725b4c73dbb890
[I 2022-03-28 10:17:38.033 ServerApp]  or http://127.0.0.1:8888/custom?token=a59dee2ea49885c518515071ab058de33e725b4c73dbb890
[I 2022-03-28 10:17:38.033 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 2022-03-28 10:17:38.082 ServerApp]

    To access the server, open this file in a browser:
        file:///home/user/.local/share/jupyter/runtime/jpserver-12891-open.html
    Or copy and paste one of these URLs:
        http://localhost:8888/custom?token=a59dee2ea49885c518515071ab058de33e725b4c73dbb890
     or http://127.0.0.1:8888/custom?token=a59dee2ea49885c518515071ab058de33e725b4c73dbb890
Opening in existing browser session.
```

### Client - frontend

The frontend code is everything that gets injected in the user's browser for building the app UI and making it interactive.

We'll first create an `templates/index.html` Jinja template file that is our main HTML entry we render in `CustomHandler`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>{{page_config['appName'] | e}}</title>
</head>
<body>

  {# Copy so we do not modify the page_config with updates. #}
  {% set page_config_full = page_config.copy() %}

  {# Set a dummy variable - we just want the side effect of the update. #}
  {% set _ = page_config_full.update(baseUrl=base_url, wsUrl=ws_url) %}

  <script id="jupyter-config-data" type="application/json">
    {{ page_config_full | tojson }}
  </script>

  <!--  bundle.js is our main JavaScript file  -->
  <script src="{{page_config['fullStaticUrl'] | e}}/bundle.js" main="index"></script>

  <script type="text/javascript">
    /* Remove token from URL. */
    (function () {
      var parsedUrl = new URL(window.location.href);
      if (parsedUrl.searchParams.get('token')) {
        parsedUrl.searchParams.delete('token');
        window.history.replaceState({ }, '', parsedUrl.href);
      }
    })();
  </script>

</body>
</html>

```

The main JavaScript file here is `static/bundle.js` located under the `static` directory. It is recommended, but not mandatory, to use **TypeScript** for the front-end code in order to get type hints in your IDE, it is the language that has been adopted for JupyterLab and many JupyterLab extensions. In this case we'll use TypeScript, transpile our code to JavaScript and use Webpack to generate the `bundle.js` final file.


## Useful Links

### Server - backend

1) Tornado - Python web framework and asynchronous networking library https://github.com/tornadoweb/tornado
2) Jinja - Template system in Python https://github.com/pallets/jinja
3) Jupyter server - core services, APIs, and REST endpoints—to Jupyter web applications https://github.com/jupyter-server/jupyter_server
4) JupyterLab server package - A set of server components for JupyterLab and JupyterLab like applications https://github.com/jupyterlab/jupyterlab_server
5)


### Client - frontend

1) Lumino - library for building interactive web applications https://github.com/jupyterlab/lumino
2) JupyterLab https://github.com/jupyterlab/jupyterlab
3) JupyterLab-contrib organization - Set of useful JupyterLab extensions https://github.com/jupyterlab-contrib
4)
