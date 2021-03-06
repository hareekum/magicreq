# magicreq

Ever had a python script that you wanted to just work on its own without worrying about whether its dependencies were installed? There are other solutions to this problem, such as packaging the script with its own [virtualenv](https://pypi.python.org/pypi/virtualenv) or using [pex](https://pypi.python.org/pypi/pex) to package everything up in a single executable binary. These solutions work well but only if you want to go through the extra effort of packaging.

magicreq gives you a solution which takes only 15 (or so) lines added to the top of your script which will create a dynamic virtualenv with your specified requirements and automatically re-run the script in that virtualenv. It even works if you haven't installed magicreq.

# Using magicreq

Always make sure to put the magicreq code above any code in the script which performs any work (usually anything other than imports). The reason for this is that magicreq does its magic by re-calling the script multiple times if needed in order to ensure that you have all of your requirements available when the actual script runs. Any code below the magicreq invocation will only ever be run once, though, so don't worry!

There are two main ways to use magicreq which only differ based on what goes into the first `try` block and what you pass to `magicreq.magic()`.

## The import method

If you have a single script you want to convert to using magicreq and don't have a requirements.txt already defined then this method should be easiest to implement. All you have to do is to put your third-party imports into the first `try` block and add a list of the required packages to the `magicreq.magic()` invocation.

```python
#!/usr/bin/env python
from __future__ import print_function
import subprocess
import sys
# put your remaining builtin imports here

try:
    # put your third-party imports here
    import requests
except ImportError:
    # Only use magicreq if this is the script being run. Can be omitted if you don't use this file
    # as a module elsewhere.
    if __name__ != '__main__':
        raise
    try:
        # If the current python environment already has magicreq, use it directly
        import magicreq
        magicreq.magic(['requests'])
    except ImportError:
        # To support environments without magicreq, this code downloads a bootstrap script which
        # will bootstrap an initial virtualenv with magicreq in it, then re-run this script within
        # the virtualenv.

        # Download the bootstrap script
        curl = subprocess.Popen(
            ['curl', '-sS', 'https://raw.githubusercontent.com/reversefold/magicreq/0.3.0/magicreq/bootstrap.py'],
            stdout=subprocess.PIPE
        )
        # pipe the bootstrap script into python
        python = subprocess.Popen([sys.executable, '-'] + sys.argv, stdin=curl.stdout)
        curl.wait()
        python.wait()
        sys.exit(curl.returncode or python.returncode)

# Your script goes here

def main():
    print(requests.get('http://example.org').text)


if __name__ == '__main__':
    main()
```

If you can be sure that your script will be running on a system with at least magicreq installed you can omit the code above that downloads and runs the bootstrap script.

```python
#!/usr/bin/env python
from __future__ import print_function
import subprocess
import sys
# put your remaining builtin imports here

try:
    # put your third-party imports here
    import requests
except ImportError:
    # Only use magicreq if this is the script being run. Can be omitted if you don't use this file
    # as a module elsewhere.
    if __name__ != '__main__':
        raise
    # If the current python environment already has magicreq, use it directly
    import magicreq
    magicreq.magic(['requests'])

# Your script goes here

def main():
    print(requests.get('http://example.org').text)


if __name__ == '__main__':
    main()
```

The imports you put in the first `try` block can be any kind of import for third party libraries, so you can put all of them in that block and not have to re-import anything below the magicreq code.
```python
try:
    import requests
    from docopt import docopt
    import boto.ec2 as boto_ec2
except ImportError:
    ...
```

## The requirements.txt method

If you have a requirements.txt already written for your project or script then the magicreq code becomes entirely boilerplate as you can use the requirements.txt to both check for installed packages and to tell magicreq what needs to be installed.

A small caveat: this method may not work if you have anything in your requirements.txt which is not a requirement (such as including `--index-url` or another option).


```python
#!/usr/bin/env python
from __future__ import print_function
import subprocess
import sys
# put your remaining builtin imports here

# Alter to point to your requirements.txt file
REQUIREMENTS = [line.strip() for line in iter(open('requirements.txt').readline, b'') if line.strip()]

# START boilerplate
try:
    import pkg_resources
    pkg_resources.require(REQUIREMENTS)

# We're expecting ImportError or pkg_resources.ResolutionError but since pkg_resources might not be importable,
# we're just catching Exception.
except Exception as exc:
    if not isinstance(exc, ImportError) and isinstance(exc, pkg_resources.VersionConflict):
        raise
    # Only use magicreq if this is the script being run. Can be omitted if you don't use this file
    # as a module elsewhere.
    if __name__ != '__main__':
        raise
    try:
        # If the current python environment already has magicreq, use it directly
        import magicreq
        magicreq.magic(REQUIREMENTS)
    except ImportError:
        # To support environments without magicreq, this code downloads a bootstrap script which
        # will bootstrap an initial virtualenv with magicreq in it, then re-run this script within
        # the virtualenv.

        # Download the bootstrap script
        curl = subprocess.Popen(
            ['curl', '-sS', 'https://raw.githubusercontent.com/reversefold/magicreq/0.3.0/magicreq/bootstrap.py'],
            stdout=subprocess.PIPE
        )
        # pipe the bootstrap script into python
        python = subprocess.Popen([sys.executable, '-'] + sys.argv, stdin=curl.stdout)
        curl.wait()
        python.wait()
        sys.exit(curl.returncode or python.returncode)
# END boilerplate

# put your third-party imports here
import requests

# Your script goes here

def main():
    print(requests.get('http://example.org').text)


if __name__ == '__main__':
    main()
```

As above, the last try/except can be reduced if you know for sure that magicreq is installed in the environment where this script will run.


# Advanced configuration

You can also configure how magicreq downloads its files. This can be useful if you have your own mirror of pypi or are using your own artifact storage service (such as S3 or Artifactory). You can even set the version of virtualenv to use for bootstrapping.

```python
...
except Exception as exc:
    if not isinstance(exc, ImportError) and isinstance(exc, pkg_resources.VersionConflict):
        raise
    # You can set any options you want to pass to pip here.
    # This example sets an alternate url for pypi to download packages.
    PIP_OPTIONS = ('--index-url http://my.artifactory.host/artifactory/api/pypi/pypi/simple '
                   '--trusted-host my.artifactory.host')

    # The base pypi URL that will be used for finding and downloading virtualenv
    PYPI_URL = 'http://my.artifactory.host/artifactory/api/pypi/pypi'

    # The version of virtualenv to use
    VENV_VERSION = '15.0.2'

    # The URL to download get-pip.py from
    GET_PIP_URL = 'http://my.artifactory.host/artifactory/static_files/get-pip.py'

    # The URL to magicreq's bootstrap.py
    MAGICREQ_BOOTSTRAP_URL = 'http://my.artifactory.host/artifactory/static_files/magicreq_bootstrap.py'

    if __name__ != '__main__':
        raise
    try:
        import magicreq
        magicreq.magic(
            REQUIREMENTS,

            # These 4 options are passed as keyword arguments here
            pip_options=PIP_OPTIONS,
            pypi_url=PYPI_URL,
            venv_version=VENV_VERSION,
            get_pip_url=GET_PIP_URL
        )
    except ImportError:
        curl = subprocess.Popen(['curl', '-sS', MAGICREQ_BOOTSTRAP_URL], stdout=subprocess.PIPE)
        python = subprocess.Popen(
            [
                sys.executable,
                '-',

                # These 4 options are passed with this special format to reduce the requirements
                # of the bootstrapping process.
                'PIP_OPTIONS:%s' % (PIP_OPTIONS,),
                'PYPI_URL:%s' % (PYPI_URL,),
                'VENV_VERSION:%s' % (VENV_VERSION,),
                'GET_PIP_URL:%s' % (GET_PIP_URL,),
            ] + sys.argv,
            stdin=curl.stdout
        )
        curl.wait()
        python.wait()
        sys.exit(curl.returncode or python.returncode)
```




Available on [pypi](https://pypi.python.org/pypi/magicreq).
