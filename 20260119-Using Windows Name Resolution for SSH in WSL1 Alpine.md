---
title: "Using Windows Name Resolution for SSH in WSL1 Alpine"
description: "When using the standard Windows SSH client together with tmux, some users occasionally see unwanted..."
pubDatetime: 2026-01-19T09:46:19.624Z
---

When using the standard Windows SSH client together with `tmux`, some users occasionally see unwanted “noise” characters appear in the terminal. One practical workaround is to run SSH from a WSL1 Alpine environment instead of using the Windows-native SSH session.

However, WSL1 Alpine has a key limitation in this setup: it cannot resolve certain hostnames (notably those relying on mDNS). To address this, the approach described here offloads hostname resolution to Windows—where resolution works correctly—and then feeds the resolved IP address back into the SSH command running inside Alpine.

## Background and Design

The solution consists of two small tools working together:

* **`resolve-proxy.c`**: runs on Windows and resolves a fully qualified domain name (FQDN) into one or more IP addresses.
* **`ssh-wrap.c`**: runs on WSL1 Alpine and wraps `ssh` so that it can transparently replace a hostname with an IP address resolved via Windows.

The combined outcome is that you can keep using WSL1 Alpine SSH (to avoid the `tmux` noise issue), while still benefiting from Windows’ ability to resolve hostnames that Alpine cannot.

## Tool 1: `resolve-proxy.c` (Windows-side resolver)

### Purpose

`resolve-proxy.c` provides a minimal command-line resolver on Windows. Given an FQDN, it prints resolved IP address(es) to standard output, which can then be consumed by other tools.

### Key Behavior

* Uses Windows networking APIs (`WSAStartup`, `getaddrinfo`) to perform DNS/name resolution.
* Supports:

  * `-4` to restrict output to IPv4
  * `-6` to restrict output to IPv6
  * `--first` to print only the first resolved address
* Prints each resolved address on its own line in a canonical textual form using `InetNtopA`.

This makes it suitable as a simple “resolver backend” that can be invoked from WSL tooling.

## Tool 2: `ssh-wrap.c` (WSL1 Alpine-side SSH wrapper)

### Purpose

`ssh-wrap.c` wraps SSH execution inside Alpine so that, if the target is a hostname (not already an IP literal), it resolves the hostname using an external resolver command and then forces SSH to connect to the resolved IP.

### How It Works

1. **Reads effective SSH configuration**

   * Executes: `ssh -G <user args...>`
   * Captures the output and extracts the final resolved SSH parameters.
   * Looks for `hostname` first (preferred), then falls back to `host`.

2. **Determines whether resolution is needed**

   * If the extracted host is already an IPv4/IPv6 literal, it is used as-is.
   * Otherwise, it resolves the hostname by calling an external resolver command.

3. **Invokes the resolver**

   * The resolver program is selected via the environment variable `SSH_HOST_RESOLVER`.
   * If not set, it defaults to `resolvehost`.
   * The wrapper runs: `<resolver> <hostname>` and captures stdout.

4. **Parses the resolver output**

   * Accepts multi-line output and selects the **first valid IP address** found.
   * Ignores blank lines and comment lines starting with `#`.
   * Validates IP strings using `inet_pton` (both IPv4 and IPv6 supported).

5. **Executes SSH with an overridden HostName**

   * Finally executes:

     * `ssh <user args...> -o HostName=<resolved_ip>`
   * This preserves the user’s SSH arguments and configuration, but forces the final network connection to use the resolved IP.

## Practical Outcome

With this setup:

* You run SSH from WSL1 Alpine (reducing the chance of terminal corruption/noise when using `tmux`).
* You still successfully connect to hosts that require Windows-style name resolution (including environments where WSL1 Alpine cannot resolve mDNS-derived names).
* The wrapper is transparent: you keep using your usual SSH invocation style, while the tool injects the necessary `HostName=<IP>` override only when required.

## Notes on Integration

This design assumes you have a Windows-accessible resolver executable available to WSL (for example, via a mounted Windows path), and that Alpine can execute it either directly or through a small shim. The key contract is simple: the resolver must print one or more IP addresses to stdout so the wrapper can select the first valid one.

## `resolve-proxy.c`

```c
#define _WIN32_WINNT 0x0600
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>
#include <string.h>

#pragma comment(lib, "Ws2_32.lib")

static void usage(const char* prog) {
    fprintf(stderr,
        "Usage: %s [ -4 | -6 ] [ --first ] <FQDN>\n"
        "  -4        IPv4 only\n"
        "  -6        IPv6 only\n"
        "  --first   Print only the first resolved address\n",
        prog
    );
}

static void print_wsa_error(const char* msg, int err) {
    fprintf(stderr, "%s (WSA error=%d)\n", msg, err);
}

int main(int argc, char** argv) {
    int family = AF_UNSPEC;   // default: both
    int first_only = 0;
    const char* fqdn = NULL;

    // Parse args
    for (int i = 1; i < argc; i++) {
        const char* a = argv[i];

        if (strcmp(a, "-4") == 0) {
            if (family == AF_INET6) {
                fprintf(stderr, "Error: -4 and -6 cannot be used together.\n");
                usage(argv[0]);
                return 2;
            }
            family = AF_INET;
        } else if (strcmp(a, "-6") == 0) {
            if (family == AF_INET) {
                fprintf(stderr, "Error: -4 and -6 cannot be used together.\n");
                usage(argv[0]);
                return 2;
            }
            family = AF_INET6;
        } else if (strcmp(a, "--first") == 0) {
            first_only = 1;
        } else if (a[0] == '-') {
            fprintf(stderr, "Error: unknown option: %s\n", a);
            usage(argv[0]);
            return 2;
        } else {
            if (fqdn != NULL) {
                fprintf(stderr, "Error: multiple hostnames provided.\n");
                usage(argv[0]);
                return 2;
            }
            fqdn = a;
        }
    }

    if (fqdn == NULL) {
        usage(argv[0]);
        return 2;
    }

    WSADATA wsa;
    int rc = WSAStartup(MAKEWORD(2, 2), &wsa);
    if (rc != 0) {
        print_wsa_error("WSAStartup failed", rc);
        return 1;
    }

    struct addrinfo hints;
    ZeroMemory(&hints, sizeof(hints));
    hints.ai_family = family;
    hints.ai_socktype = SOCK_STREAM;  // typical; not strictly required
    hints.ai_protocol = IPPROTO_TCP;

    struct addrinfo* res = NULL;
    rc = getaddrinfo(fqdn, NULL, &hints, &res);
    if (rc != 0) {
        fprintf(stderr, "getaddrinfo failed for '%s' (err=%d)\n", fqdn, rc);
        WSACleanup();
        return 1;
    }

    int printed = 0;
    for (struct addrinfo* p = res; p != NULL; p = p->ai_next) {
        char buf[INET6_ADDRSTRLEN];
        void* addr = NULL;

        if (p->ai_family == AF_INET) {
            struct sockaddr_in* sa = (struct sockaddr_in*)p->ai_addr;
            addr = &(sa->sin_addr);
        } else if (p->ai_family == AF_INET6) {
            struct sockaddr_in6* sa6 = (struct sockaddr_in6*)p->ai_addr;
            addr = &(sa6->sin6_addr);
        } else {
            continue;
        }

        if (InetNtopA(p->ai_family, addr, buf, (DWORD)sizeof(buf)) == NULL) {
            print_wsa_error("InetNtopA failed", WSAGetLastError());
            continue;
        }

        puts(buf);
        printed++;

        if (first_only) {
            break;
        }
    }

    freeaddrinfo(res);
    WSACleanup();

    if (printed == 0) {
        fprintf(stderr, "No addresses found for '%s'\n", fqdn);
        return 1;
    }

    return 0;
}
```

## `ssh-wrap.c`

```c
// sshwrap.c
#define _POSIX_C_SOURCE 200809L
#include <arpa/inet.h>
#include <errno.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/wait.h>
#include <unistd.h>

static void die(const char *msg) {
    perror(msg);
    exit(1);
}

static void *xmalloc(size_t n) {
    void *p = malloc(n);
    if (!p) { fprintf(stderr, "malloc failed\n"); exit(1); }
    return p;
}

static char *trim_inplace(char *s) {
    while (*s == ' ' || *s == '\t' || *s == '\r' || *s == '\n') s++;
    size_t len = strlen(s);
    while (len > 0) {
        char c = s[len - 1];
        if (c == ' ' || c == '\t' || c == '\r' || c == '\n') {
            s[len - 1] = '\0';
            len--;
        } else break;
    }
    return s;
}

static bool is_ip_literal(const char *s) {
    unsigned char buf[16];
    if (inet_pton(AF_INET, s, buf) == 1) return true;
    if (inet_pton(AF_INET6, s, buf) == 1) return true;
    return false;
}

static char *run_capture_stdout(char *const argv[]) {
    int p[2];
    if (pipe(p) != 0) die("pipe");

    pid_t pid = fork();
    if (pid < 0) die("fork");

    if (pid == 0) {
        // child
        close(p[0]);
        if (dup2(p[1], STDOUT_FILENO) < 0) die("dup2");
        close(p[1]);
        execvp(argv[0], argv);
        // exec failed
        fprintf(stderr, "execvp failed: %s\n", argv[0]);
        _exit(127);
    }

    // parent
    close(p[1]);
    size_t cap = 8192;
    size_t len = 0;
    char *buf = (char *)xmalloc(cap);
    for (;;) {
        if (len + 4096 + 1 > cap) {
            cap *= 2;
            buf = (char *)realloc(buf, cap);
            if (!buf) { fprintf(stderr, "realloc failed\n"); exit(1); }
        }
        ssize_t r = read(p[0], buf + len, 4096);
        if (r < 0) {
            if (errno == EINTR) continue;
            die("read");
        }
        if (r == 0) break;
        len += (size_t)r;
    }
    close(p[0]);
    buf[len] = '\0';

    int status = 0;
    if (waitpid(pid, &status, 0) < 0) die("waitpid");
    if (!WIFEXITED(status) || WEXITSTATUS(status) != 0) {
        fprintf(stderr, "child exited abnormally (status=%d)\n", status);
        free(buf);
        return NULL;
    }
    return buf;
}

static char *extract_value_from_sshG(const char *text, const char *key) {
    // key is expected like "hostname" or "host"
    size_t keylen = strlen(key);

    const char *p = text;
    while (*p) {
        // get one line
        const char *line = p;
        const char *nl = strchr(p, '\n');
        size_t linelen = nl ? (size_t)(nl - p) : strlen(p);

        // match: "<key><space...><value...>"
        if (linelen > keylen && strncmp(line, key, keylen) == 0) {
            char c = line[keylen];
            if (c == ' ' || c == '\t') {
                // copy value part
                const char *v = line + keylen;
                while (*v == ' ' || *v == '\t') v++;
                if ((size_t)(v - line) < linelen) {
                    size_t vlen = linelen - (size_t)(v - line);
                    char *out = (char *)xmalloc(vlen + 1);
                    memcpy(out, v, vlen);
                    out[vlen] = '\0';
                    return trim_inplace(out);
                }
            }
        }

        p = nl ? nl + 1 : p + linelen;
    }
    return NULL;
}

static char *first_ip_from_multiline_output(char *out) {
    // out is a writable buffer (we modify it in-place)
    // returns strdup()'d IP string or NULL
    char *p = out;

    while (*p) {
        // 1) get one line [p, eol)
        char *line = p;
        char *eol = strchr(p, '\n');
        if (eol) {
            *eol = '\0';
            p = eol + 1;
        } else {
            p = p + strlen(p);
        }

        // 2) trim line
        line = trim_inplace(line);
        if (*line == '\0') continue;
        if (*line == '#') continue; // allow comments

        // 3) take first token in the line
        char *tok = line;
        char *sp = tok;
        while (*sp && *sp != ' ' && *sp != '\t' && *sp != '\r') sp++;
        *sp = '\0';

        // 4) validate as IP literal
        if (is_ip_literal(tok)) {
            char *ip = strdup(tok);
            if (!ip) { fprintf(stderr, "strdup failed\n"); exit(1); }
            return ip;
        }
    }
    return NULL;
}

static char *resolve_to_ip(const char *resolver, const char *hostname) {
    // resolver hostname -> stdout => multiple lines, possibly multiple IPs
    char *const argv[] = { (char *)resolver, (char *)hostname, NULL };
    char *out = run_capture_stdout(argv);
    if (!out) return NULL;

    char *ip = first_ip_from_multiline_output(out);
    if (!ip) {
        fprintf(stderr, "resolver produced no valid IPs\n");
        free(out);
        return NULL;
    }

    free(out);
    return ip;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <ssh-args...>\n", argv[0]);
        fprintf(stderr, "Example: %s user@host -p 22\n", argv[0]);
        return 2;
    }

    // 1) Build argv for: ssh -G <user args...>
    // size: "ssh", "-G", (argc-1 user args), NULL
    int gN = 2 + (argc - 1) + 1;
    char **gargv = (char **)xmalloc(sizeof(char*) * (size_t)gN);
    int gi = 0;
    gargv[gi++] = (char *)"ssh";
    gargv[gi++] = (char *)"-G";
    for (int i = 1; i < argc; i++) gargv[gi++] = argv[i];
    gargv[gi++] = NULL;

    char *cfg = run_capture_stdout(gargv);
    free(gargv);

    if (!cfg) {
        fprintf(stderr, "failed to run 'ssh -G'\n");
        return 1;
    }

    // 2) Extract hostname (prefer "hostname", fallback "host")
    char *host = extract_value_from_sshG(cfg, "hostname");
    if (!host) host = extract_value_from_sshG(cfg, "host");
    free(cfg);

    if (!host || *host == '\0') {
        fprintf(stderr, "could not extract hostname from 'ssh -G' output\n");
        free(host);
        return 1;
    }

    // 3) Resolve to IP (skip if already IP)
    char *ip = NULL;
    if (is_ip_literal(host)) {
        ip = strdup(host);
        if (!ip) { fprintf(stderr, "strdup failed\n"); exit(1); }
    } else {
        const char *resolver = getenv("SSH_HOST_RESOLVER");
        if (!resolver || *resolver == '\0') resolver = "resolvehost";
        ip = resolve_to_ip(resolver, host);
        if (!ip) {
            fprintf(stderr, "failed to resolve host '%s'\n", host);
            free(host);
            return 1;
        }
    }

    free(host);

    // 4) Exec: ssh -o HostName=<IP> <user args...>
    // size: "ssh", (argc-1 user args), "-o", "HostName=<ip>", NULL
    char optbuf[256];
    if (snprintf(optbuf, sizeof(optbuf), "HostName=%s", ip) >= (int)sizeof(optbuf)) {
        fprintf(stderr, "IP string too long\n");
        free(ip);
        return 1;
    }
    free(ip);

    int eN = 1 + (argc - 1) + 2 + 1 + 1; // ssh + (-o + opt) + userargs + NULL
    char **eargv = (char **)xmalloc(sizeof(char*) * (size_t)eN);
    int ei = 0;
    eargv[ei++] = (char *)"ssh";
    eargv[ei++] = (char *)"-o";
    eargv[ei++] = optbuf;
    for (int i = 1; i < argc; i++) eargv[ei++] = argv[i];
    eargv[ei++] = NULL;

    execvp("ssh", eargv);
    die("execvp ssh");
    return 127;
}
```
