+++
title = "SUD 01: Understanding the usage of UNIX Socket in SUD to avoid the use of setuid"
date = "2025-02-08"
+++

The Super User Daemon (SUD) utilizes UNIX domain sockets to allow unprivileged clients to communicate with the 
privileged daemon. This communication enables the clients to request elevated privileges securely, ensuring 
controlled access to administrative functions without granting direct root access.

# Architecture of SUD

SUD follows a simple yet effective architecture. It consists of a privileged daemon running with root permissions, 
which listens for incoming connections over a UNIX socket. When a client connects to this socket, the daemon 
authenticates the user and, if authorized, executes the requested command with elevated privileges.

This architecture eliminates the need for setuid binaries, reducing security risks associated with privilege escalation.
Since the client process itself does not require elevated permissions, it simply connects to the SUD daemon over the 
UNIX domain socket. The daemon then decides whether to grant or deny the request based on authentication and policy 
enforcement. This approach significantly improves security by centralizing privilege management within a controlled 
and audited environment.

# Systemd socket activation

SUD is designed to work with systemd's socket activation mechanism. This means that instead of having a daemon running 
persistently in the background, systemd listens for connections and only starts SUD when a request is received. 

The [sud.socket](https://github.com/ErnyTech/sud/blob/master/systemd/sud.socket) file defines how systemd should handle 
socket activation for SUD:
```
[Unit]
Description=Super User Daemon - privilege manager for systemd/Linux 
Requires=sud@.service

[Socket]
ListenSequentialPacket=@sud_privilege_manager_socket
Accept=yes

[Install]
WantedBy=sockets.target
```

The corresponding service unit file 
[sud@.service.template](https://github.com/ErnyTech/sud/blob/master/systemd/sud%40.service.template) execute SUD when a 
connection is established:
```
[Unit]
Description=Super User Daemon - privilege manager for systemd/Linux
PartOf=sud.socket

[Service]
Type=exec
ExecStart=${SUD_BIN} --daemon
```

When a client attempts to connect to `@sud_privilege_manager_socket`, systemd:
1. Accepts the connection.
2. Spawns an instance of `sud@.service`.
3. Passes the file descriptor (FD) of the connection to the new service instance.
4. The SUD daemon instance reads the connection, processes the request, and then terminates.

# Server implementation

The handling of UNIX sockets passed by systemd is implemented in the 
[server.c](https://github.com/ErnyTech/sud/blob/master/src/server.c) file, this code retrieves the file descriptors 
passed by systemd:
```
[...]

num_fds = sd_listen_fds_with_names(0, &names);
if (num_fds < 0) {
  SUD_DEBUG_ERRNO();
  return 1;
}

if (num_fds == 0 || names == nullptr) {
  SUD_ERR("Unable to find any file descriptors\n");
  return 1;
}

for (int i = 0; i < num_fds; i++) {
  if (strcmp(names[i], "connection") == 0) {
    conn_fd = i + SD_LISTEN_FDS_START;
    break;
  }
}

[...]

if (!sd_is_socket(conn_fd, AF_UNIX, SOCK_SEQPACKET, 0)) {
  SUD_ERR("Wrong socket type\n");
  return 1;
}    
```
Now that SUD has obtained the FD of the UNIX socket connection it can process the authentication and execution of the 
request.

# Client implementation

The client implementation of SUD ([client.c](https://github.com/ErnyTech/sud/blob/master/src/client.c)) is very minimal, 
essentially it is a simple connection to UNIX socket and receiving a struct containing the response from the daemon:

```
int unix_socket_connect(const char *socket_path, size_t socket_path_len, int timeout) {
  int sock_fd;
  int rc;
  struct sockaddr_un sock_addr;

  memset(&sock_addr, 0, sizeof(sock_addr));
  sock_addr.sun_family = AF_UNIX;
  memcpy(sock_addr.sun_path, socket_path, socket_path_len);

  sock_fd = socket(sock_addr.sun_family, SOCK_SEQPACKET, 0);
    if (sock_fd < 0) {
      SUD_DEBUG_ERRNO_CLIENT();
      return -1;
  }

  rc = -1;
  while (timeout-- >= 0) {
      rc = connect(sock_fd, (struct sockaddr *)&sock_addr, offsetof(struct sockaddr_un, sun_path) + socket_path_len);
      if (rc == 0) {
          break;
      } else {
          sleep(1);
      }
  }

  if (rc < 0) {
      SUD_DEBUG_ERRNO_CLIENT();
      close(sock_fd);
      return -1;
  }

  return sock_fd;
}

int main_client() {
  [...]
  
  fd = unix_socket_connect(SUD_SOCKET_PATH, sizeof(SUD_SOCKET_PATH) - 1, 1);
  
  if (recv(fd, &msg, sizeof(struct sud_response_msg), 0) != sizeof(struct sud_response_msg)) {
      SUD_DEBUG_ERRNO_CLIENT();
      msg.error = SUD_MSG_ERROR_RESPONSE_FAIL;
  }

  if (msg.error != 0) {
      reset_term();
      fprintf(stderr, "error %d\n", msg.error);
  }

  return msg.exit_code;
}
```

Note how the client does not send anything to the server, in fact the SUD architecture does not require any 
communication from the client other than the connection to the UNIX socket. 

Authentication and obtaining the cmdline to execute is managed only by the privileged process which "steals" this
information from the non-privileged process, in this way the non-privileged process is totally managed as untrusted and
does not participate in any relevant operation in the management of permissions.

The only communication via the UNIX socket connection is sent from the server to the client and contains SUD internal 
error code and the exit code of the executed binary.

# Simplifying diagram of a connection with SUD

<p>
  <img id="sud_connection_diagram-image" src="/articles/sud_01/sud_connection_diagram.svg">
</p>

<script>
  function updateImage() {
      const img = document.getElementById("sud_connection_diagram-image");
      const isDarkMode = document.documentElement.getAttribute("data-theme") === "dark";
      img.src = isDarkMode ? "/articles/sud_01/sud_connection_diagram_dark.svg" : "/articles/sud_01/sud_connection_diagram.svg";
  }

  const observer = new MutationObserver(updateImage);
  observer.observe(document.documentElement, { attributes: true, attributeFilter: ["data-theme"] });
  updateImage();
</script>

# Conclusion

SUD's socket-based communication, combined with systemd socket activation, provides a robust, efficient, and secure way 
to manage privileged operations in Linux. By leveraging UNIX domain sockets and systemd’s security features, SUD 
ensures controlled access to privileged operations while minimizing the risks associated with long-running daemons.

The next article will focus on how SUD proceeds to authenticate users without requiring any action on the part 
from the client.
