#!/usr/bin/env python

"""
Launch Jupyter Notebook within a Docker notebook image and
automatically open up the URL within the default web browser.
"""

# Author: Xiangmin Jiao <xmjiao@gmail.com>

from __future__ import print_function  # Only Python 2.x

import argparse
import sys
import time

# Process command-line arguments
parser = argparse.ArgumentParser(description=__doc__)

parser.add_argument('-u', "--user",
                    help='username used by the image. The default is compdatasci.',
                    default="compdatasci")

parser.add_argument('-i', '--image',
                    help='The Docker image to use The default is compdatasci/scipy-jupyter.',
                    default="compdatasci/scipy-jupyter")

parser.add_argument('-p', '--pull',
                    help='Pull the latest Docker image. The default is not to pull.',
                    dest='pull', action='store_true')

parser.set_defaults(pull=False)

parser.add_argument('notebook', nargs='?',
                    help='The notebook to open.', default="")

args = parser.parse_args()
image = args.image
user = args.user
notebook = args.notebook
pull = args.pull


def random_ports(port, n):
    """Generate a list of n random ports near the given port.

    The first 5 ports will be sequential, and the remaining n-5 will be
    randomly selected in the range [port-2*n, port+2*n].
    """
    import random

    for i in range(min(5, n)):
        yield port + i
    for i in range(n - 5):
        yield max(1, port + random.randint(-2 * n, 2 * n))


def id_generator(size=6):
    """Generate a container ID"""
    import random
    import string

    chars = string.ascii_uppercase + string.digits
    return "desktop_" + (''.join(random.choice(chars) for _ in range(size)))


def find_free_port(port, retries):
    "Find a free port"
    import socket

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    for port in random_ports(port, retries + 1):
        try:
            sock.bind(("127.0.0.1", port))
            sock.close()
            return port
        except socket.error as e:
            continue

    print("Error: Could not find a free port.")
    sys.exit(-1)


def wait_net_service(port, timeout=30):
    """ Wait for network service to appear.
    """
    import socket

    for i in range(timeout * 10):
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.connect(("127.0.0.1", port))
        except socket.error as err:
            sock.close()
            time.sleep(0.1)
            continue
        else:
            sock.close()
            time.sleep(1)
            return True


if __name__ == "__main__":
    import os
    import subprocess
    import webbrowser
    import platform

    pwd = os.getcwd()

    if pull or not subprocess.check_output(['docker', 'images', '-q', image]):
        try:
            err = subprocess.call(["docker", "pull", image])
        except BaseException:
            err = -1

        if err:
            sys.exit(err)

    # Generate a container ID and find an unused port
    container = id_generator()
    port = str(find_free_port(8888, 50))

    homedir = os.path.expanduser('~')
    if platform.system() == "Linux":
        uid = str(os.getuid())
    else:
        uid = ""

    print("Starting up docker image...")

    # Start the docker image in the background and pipe the stderr
    subprocess.call(["docker", "run", "-d", "--rm", "--name", container,
                     "-w", "/home/" + user + "/shared",
                     "-v", pwd + ":/home/" + user + "/shared",
                     "-p", "127.0.0.1:" + port + ":" + port,
                     "--env", "HOST_UID=" + uid, image,
                     "jupyter-notebook --no-browser --ip=0.0.0.0 --port " + port +
                     ">> /home/" + user + "/.log/jupyter.log 2>&1"])

    wait_for_url = True

    # Wait for user to press Ctrl-C
    while True:
        try:
            if wait_for_url:
                # Wait until the file is not empty
                while not subprocess.check_output(["docker", "exec", container,
                                                   "cat", "/home/" + user + "/.log/jupyter.log"]):
                    time.sleep(1)

                p = subprocess.Popen(["docker", "exec", container,
                                      "tail", "-F", "/home/" + user + "/.log/jupyter.log"],
                                     stdout=subprocess.PIPE, stderr=subprocess.PIPE,
                                     universal_newlines=True)

                # Monitor the stdout to extract the URL
                for stdout_line in iter(p.stdout.readline, ""):
                    ind = stdout_line.find("http://0.0.0.0:")

                    if ind >= 0:
                        # Open browser if found URL
                        if not notebook:
                            url = "http://localhost:" + \
                                stdout_line[ind + 15:-1]
                        else:
                            url = "http://localhost:" + port + "/notebooks/" + notebook + \
                                stdout_line[stdout_line.find("?token="):-1]

                        print("Copy/paste this URL into your browser when you connect for the first time:")
                        print("    ", url)

                        wait_net_service(int(port))
                        webbrowser.open(url)
                        p.stdout.close()
                        p.terminate()
                        wait_for_url = False
                        break

            print("Press Ctrl-C to stop the server.")
            while True:
                time.sleep(3600)
        except subprocess.CalledProcessError:
            time.sleep(1)
            continue
        except KeyboardInterrupt:
            try:
                print("Type Y or Ctrl-C again to stop the server: ")
                yes = sys.stdin.read(1)
            except KeyboardInterrupt:
                yes = "Y"
            finally:
                if yes == "Y" or yes == "y":
                    print('*** Stopping the server.')
                    p = subprocess.Popen(["docker", "stop", container],
                                         stdout=subprocess.PIPE, stderr=subprocess.PIPE)
                    out, err = p.communicate()
                    sys.exit(0)
                else:
                    print('Invalid response. Resuming...')
        except OSError:
            system.exit(-1)
