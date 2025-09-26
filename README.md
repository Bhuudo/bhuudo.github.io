# bhuudo.github.io
<pre></pre>
// frebcord.c
// Minimal C Discord terminal client using OpenSSL, raw WebSocket, and embedded jsmn.
// Build: gcc -std=c11 discord_cli.c -lssl -lcrypto -lpthread -o discord_cli
// Run:   DISCORD_BOT_TOKEN="Bot YOUR_TOKEN" ./discord_cli
//
// Notes:
// - This is intentionally minimal: no full HTTP/WS library, no robust error handling.
// - It prints incoming messages and heartbeats, enough to prove the loop works.
// - Windows (MinGW): replace sockets with Winsock2 init; OpenSSL works with MinGW.
// - Intents are limited to GUILDS + GUILD_MESSAGES (513). Adjust as needed.

#define DISCORD_BOT_TOKEN "Bot MTIzNjgxNDMyNjUwMTc0MDYwNQ.GOByzS.WGXcUKF6noThTZnkPQFkCG1ReIBTjBZ9XvCg70"

#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdarg.h>
#include <time.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <netdb.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/rand.h>
#include <openssl/sha.h>
#include <openssl/bio.h>

// --------------------- Tiny JSON parser (jsmn) ---------------------
// Embedded jsmn (MIT) – minimal JSON tokenizer (objects, arrays, strings, primitives).
// Source: https://github.com/zserge/jsmn (inlined for single-file simplicity)

typedef enum { JSMN_UNDEFINED = 0, JSMN_OBJECT = 1, JSMN_ARRAY = 2, JSMN_STRING = 3, JSMN_PRIMITIVE = 4 } jsmntype_t;

typedef struct {
    jsmntype_t type;
    int start;
    int end;
    int size;
#ifdef JSMN_PARENT_LINKS
    int parent;
#endif
} jsmntok_t;

typedef struct {
    unsigned int pos;     /* offset in the JSON string */
    unsigned int toknext; /* next token to allocate */
    int toksuper;         /* superior token node, e.g parent object or array */
} jsmn_parser;

static void jsmn_init(jsmn_parser *parser) { parser->pos = 0; parser->toknext = 0; parser->toksuper = -1; }

static jsmntok_t *jsmn_alloc_token(jsmn_parser *parser, jsmntok_t *tokens, size_t num_tokens) {
    if (parser->toknext >= num_tokens) return NULL;
    jsmntok_t *tok = &tokens[parser->toknext++];
    tok->start = tok->end = -1; tok->size = 0; tok->type = JSMN_UNDEFINED;
#ifdef JSMN_PARENT_LINKS
    tok->parent = -1;
#endif
    return tok;
}

static void jsmn_fill_token(jsmntok_t *token, jsmntype_t type, int start, int end) {
    token->type = type; token->start = start; token->end = end; token->size = 0;
}

static int jsmn_parse(jsmn_parser *parser, const char *js, size_t len, jsmntok_t *tokens, unsigned int num_tokens) {
    int count = parser->toknext; int i; jsmntok_t *tok;
    for (; parser->pos < len; parser->pos++) {
        char c = js[parser->pos];
        switch (c) {
            case '{': case '[':
                count++;
                tok = jsmn_alloc_token(parser, tokens, num_tokens);
                if (tok == NULL) return -1;
                tok->type = (c == '{' ? JSMN_OBJECT : JSMN_ARRAY);
                tok->start = parser->pos;
#ifdef JSMN_PARENT_LINKS
                tok->parent = parser->toksuper;
#endif
                parser->toksuper = parser->toknext - 1;
                break;
            case '}': case ']':
                for (i = parser->toknext - 1; i >= 0; i--) {
                    tok = &tokens[i];
                    if (tok->start != -1 && tok->end == -1) {
                        tok->end = parser->pos + 1;
                        parser->toksuper = -1;
                        break;
                    }
                }
                break;
            case '\"': {
                int start = parser->pos;
                parser->pos++;
                for (; parser->pos < len; parser->pos++) {
                    c = js[parser->pos];
                    if (c == '\"') {
                        tok = jsmn_alloc_token(parser, tokens, num_tokens);
                        if (tok == NULL) return -1;
                        jsmn_fill_token(tok, JSMN_STRING, start + 1, parser->pos);
#ifdef JSMN_PARENT_LINKS
                        tok->parent = parser->toksuper;
#endif
                        break;
                    }
                    if (c == '\\') parser->pos++;
                }
                break;
            }
            case '\t': case '\r': case '\n': case ' ': case ',' : case ':' :
                break;
            default: {
                int start = parser->pos;
                while (parser->pos < len) {
                    c = js[parser->pos];
                    if (c == '\t' || c == '\r' || c == '\n' || c == ' ' || c == ',' || c == ']' || c == '}')
                        break;
                    parser->pos++;
                }
                tok = jsmn_alloc_token(parser, tokens, num_tokens);
                if (tok == NULL) return -1;
                jsmn_fill_token(tok, JSMN_PRIMITIVE, start, parser->pos);
#ifdef JSMN_PARENT_LINKS
                tok->parent = parser->toksuper;
#endif
                parser->pos--;
                break;
            }
        }
    }
    return parser->toknext;
}

static int js_eq(const char *js, const jsmntok_t *t, const char *s) {
    int len = t->end - t->start;
    return (t->type == JSMN_STRING && (int)strlen(s) == len && strncmp(js + t->start, s, len) == 0);
}

static int js_get_int(const char *js, const jsmntok_t *t) {
    char buf[32]; int len = t->end - t->start; if (len >= (int)sizeof(buf)) return 0;
    memcpy(buf, js + t->start, len); buf[len] = 0; return atoi(buf);
}

static char *js_get_strdup(const char *js, const jsmntok_t *t) {
    int len = t->end - t->start;
    char *s = (char*)malloc(len + 1);
    memcpy(s, js + t->start, len); s[len] = 0; return s;
}

// --------------------- Utility: base64 ---------------------

static char *base64_encode(const unsigned char *data, size_t input_length) {
    BIO *bio, *b64; BUF_MEM *buffer_ptr;
    b64 = BIO_new(BIO_f_base64());
    bio = BIO_new(BIO_s_mem());
    b64 = BIO_push(b64, bio);
    BIO_set_flags(b64, BIO_FLAGS_BASE64_NO_NL);
    BIO_write(b64, data, (int)input_length);
    BIO_flush(b64);
    BIO_get_mem_ptr(b64, &buffer_ptr);
    char *out = (char*)malloc(buffer_ptr->length + 1);
    memcpy(out, buffer_ptr->data, buffer_ptr->length);
    out[buffer_ptr->length] = 0;
    BIO_free_all(b64);
    return out;
}

// --------------------- Networking + TLS ---------------------

typedef struct {
    int fd;
    SSL *ssl;
    SSL_CTX *ctx;
} tls_conn;

static int tcp_connect(const char *host, const char *port) {
    struct addrinfo hints; memset(&hints, 0, sizeof(hints));
    hints.ai_socktype = SOCK_STREAM; hints.ai_family = AF_UNSPEC;
    struct addrinfo *res;
    int rv = getaddrinfo(host, port, &hints, &res);
    if (rv != 0) { fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv)); return -1; }

    int fd = -1;
    for (struct addrinfo *p = res; p; p = p->ai_next) {
        fd = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (fd == -1) continue;
        if (connect(fd, p->ai_addr, p->ai_addrlen) == 0) { break; }
        close(fd); fd = -1;
    }
    freeaddrinfo(res);
    return fd;
}

static int tls_handshake(tls_conn *c, const char *host) {
    SSL_library_init();
    SSL_load_error_strings();
    OpenSSL_add_all_algorithms();

    c->ctx = SSL_CTX_new(TLS_client_method());
    if (!c->ctx) return -1;
    SSL_CTX_set_min_proto_version(c->ctx, TLS1_2_VERSION);

    // SNI
    c->ssl = SSL_new(c->ctx);
    if (!c->ssl) return -1;
    SSL_set_tlsext_host_name(c->ssl, host);
    SSL_set_fd(c->ssl, c->fd);
    int ret = SSL_connect(c->ssl);
    if (ret != 1) {
        fprintf(stderr, "SSL_connect failed: %d\n", SSL_get_error(c->ssl, ret));
        return -1;
    }
    return 0;
}

static int tls_write(tls_conn *c, const void *buf, size_t len) {
    int n = SSL_write(c->ssl, buf, (int)len);
    return n;
}

static int tls_read(tls_conn *c, void *buf, size_t len) {
    int n = SSL_read(c->ssl, buf, (int)len);
    return n;
}

static void tls_close(tls_conn *c) {
    if (c->ssl) { SSL_shutdown(c->ssl); SSL_free(c->ssl); }
    if (c->ctx) { SSL_CTX_free(c->ctx); }
    if (c->fd >= 0) { close(c->fd); }
}

// --------------------- Minimal WebSocket ---------------------

static int ws_handshake(tls_conn *c, const char *host, const char *path, char *sec_key_out, size_t sec_key_out_sz) {
    unsigned char nonce[16]; RAND_bytes(nonce, sizeof(nonce));
    char *sec_key = base64_encode(nonce, sizeof(nonce));

    char req[1024];
    int n = snprintf(req, sizeof(req),
        "GET %s HTTP/1.1\r\n"
        "Host: %s\r\n"
        "Upgrade: websocket\r\n"
        "Connection: Upgrade\r\n"
        "Sec-WebSocket-Key: %s\r\n"
        "Sec-WebSocket-Version: 13\r\n"
        "User-Agent: c-cli/1.0\r\n"
        "\r\n",
        path, host, sec_key);
    if (n <= 0 || n >= (int)sizeof(req)) { free(sec_key); return -1; }

    if (tls_write(c, req, n) <= 0) { free(sec_key); return -1; }

    // Read simple HTTP response headers
    char buf[4096]; int total = 0;
    while (total < (int)sizeof(buf) - 1) {
        int r = tls_read(c, buf + total, sizeof(buf) - 1 - total);
        if (r <= 0) break;
        total += r;
        buf[total] = 0;
        if (strstr(buf, "\r\n\r\n")) break;
    }
    if (strncmp(buf, "HTTP/1.1 101", 12) != 0) { fprintf(stderr, "WS upgrade failed:\n%s\n", buf); free(sec_key); return -1; }

    // Return Sec-WebSocket-Key (if needed)
    strncpy(sec_key_out, sec_key, sec_key_out_sz - 1);
    sec_key_out[sec_key_out_sz - 1] = 0;
    free(sec_key);
    return 0;
}

static int ws_send_text(tls_conn *c, const char *text) {
    size_t len = strlen(text);
    unsigned char header[10]; size_t hlen = 0;
    header[0] = 0x81; // FIN=1, opcode=1 (text)
    unsigned char mask_key[4]; RAND_bytes(mask_key, sizeof(mask_key));
    if (len <= 125) {
        header[1] = 0x80 | (unsigned char)len; hlen = 2;
    } else if (len <= 65535) {
        header[1] = 0x80 | 126; header[2] = (len >> 8) & 0xFF; header[3] = len & 0xFF; hlen = 4;
    } else {
        header[1] = 0x80 | 127;
        for (int i = 0; i < 8; i++) header[2 + i] = (unsigned char)((len >> (8*(7-i))) & 0xFF);
        hlen = 10;
    }

    // Append mask key
    unsigned char *frame = (unsigned char*)malloc(hlen + 4 + len);
    memcpy(frame, header, hlen);
    memcpy(frame + hlen, mask_key, 4);
    for (size_t i = 0; i < len; i++) {
        frame[hlen + 4 + i] = ((unsigned char)text[i]) ^ mask_key[i % 4];
    }
    int rc = tls_write(c, frame, hlen + 4 + len);
    free(frame);
    return rc;
}

static int ws_read_frame(tls_conn *c, unsigned char **payload_out, size_t *len_out, unsigned char *opcode_out) {
    unsigned char hdr[2];
    int r = tls_read(c, hdr, 2);
    if (r <= 0) return -1;
    unsigned char fin = (hdr[0] & 0x80) != 0;
    unsigned char opcode = hdr[0] & 0x0F;
    unsigned char mask = (hdr[1] & 0x80) != 0;
    uint64_t len = hdr[1] & 0x7F;

    if (len == 126) {
        unsigned char ext[2]; if (tls_read(c, ext, 2) != 2) return -1;
        len = ((uint64_t)ext[0] << 8) | (uint64_t)ext[1];
    } else if (len == 127) {
        unsigned char ext[8]; if (tls_read(c, ext, 8) != 8) return -1;
        len = 0; for (int i = 0; i < 8; i++) { len = (len << 8) | ext[i]; }
    }

    unsigned char mask_key[4] = {0};
    if (mask) { if (tls_read(c, mask_key, 4) != 4) return -1; }

    unsigned char *payload = (unsigned char*)malloc(len + 1);
    size_t got = 0;
    while (got < len) {
        int n = tls_read(c, payload + got, len - got);
        if (n <= 0) { free(payload); return -1; }
        got += n;
    }
    if (mask) { for (size_t i = 0; i < len; i++) payload[i] ^= mask_key[i % 4]; }
    payload[len] = 0;

    *payload_out = payload; *len_out = len; *opcode_out = opcode;
    (void)fin;
    return 0;
}

// --------------------- Discord protocol helpers ---------------------

static char *json_build_identify(const char *bot_token) {
    // Minimal identify with intents: GUILDS (1) + GUILD_MESSAGES (512) = 513
    const char *tmpl =
        "{\"op\":2,\"d\":{"
        "\"token\":\"%s\","
        "\"intents\":513,"
        "\"properties\":{"
        "\"os\":\"linux\",\"browser\":\"c-cli\",\"device\":\"c-cli\""
        "}}}";
    size_t need = strlen(tmpl) + strlen(bot_token) + 1;
    char *out = (char*)malloc(need);
    snprintf(out, need, tmpl, bot_token);
    return out;
}

static char *json_build_heartbeat(int last_seq) {
    char buf[64];
    if (last_seq >= 0) snprintf(buf, sizeof(buf), "{\"op\":1,\"d\":%d}", last_seq);
    else snprintf(buf, sizeof(buf), "{\"op\":1,\"d\":null}");
    return strdup(buf);
}

typedef struct {
    tls_conn conn;
    volatile int heartbeat_interval_ms;
    volatile int last_seq;
    volatile int running;
} runtime_t;

static void *heartbeat_thread(void *arg) {
    runtime_t *rt = (runtime_t*)arg;
    while (rt->running) {
        if (rt->heartbeat_interval_ms > 0) {
            char *hb = json_build_heartbeat(rt->last_seq);
            ws_send_text(&rt->conn, hb);
            free(hb);
            // printf(">> HEARTBEAT (s=%d)\n", rt->last_seq);
            usleep(rt->heartbeat_interval_ms * 1000);
        } else {
            usleep(200 * 1000);
        }
    }
    return NULL;
}

static void handle_dispatch(const char *json, jsmntok_t *toks, int ntoks, runtime_t *rt) {
    // We need "t", "s", and for MESSAGE_CREATE: d.content, d.author.username, d.channel_id
    const jsmntok_t *t_field = NULL, *s_field = NULL, *d_obj = NULL;
    for (int i = 1; i < ntoks; i++) {
        if (js_eq(json, &toks[i], "t")) t_field = &toks[i+1];
        else if (js_eq(json, &toks[i], "s")) s_field = &toks[i+1];
        else if (js_eq(json, &toks[i], "d")) d_obj = &toks[i+1];
    }
    if (s_field && s_field->type == JSMN_PRIMITIVE) rt->last_seq = js_get_int(json, s_field);
    if (!t_field || t_field->type != JSMN_STRING || !d_obj) return;

    char *event = js_get_strdup(json, t_field);
    if (strcmp(event, "READY") == 0) {
        // Print username
        // Find d.user.username
        char *username = NULL;
        for (int i = 1; i < ntoks; i++) {
            if (toks[i].start >= d_obj->start && toks[i].end <= d_obj->end) {
                if (js_eq(json, &toks[i], "user")) {
                    jsmntok_t user = toks[i+1];
                    for (int j = i+1; j < ntoks; j++) {
                        if (toks[j].start >= user.start && toks[j].end <= user.end && js_eq(json, &toks[j], "username")) {
                            username = js_get_strdup(json, &toks[j+1]); break;
                        }
                    }
                    break;
                }
            }
        }
        if (username) { printf("READY: logged in as %s\n", username); free(username); }
        else { printf("READY\n"); }
    } else if (strcmp(event, "MESSAGE_CREATE") == 0) {
        char *content = NULL, *author = NULL, *channel_id = NULL;
        for (int i = 1; i < ntoks; i++) {
            if (toks[i].start >= d_obj->start && toks[i].end <= d_obj->end) {
                if (js_eq(json, &toks[i], "content")) content = js_get_strdup(json, &toks[i+1]);
                else if (js_eq(json, &toks[i], "channel_id")) channel_id = js_get_strdup(json, &toks[i+1]);
                else if (js_eq(json, &toks[i], "author")) {
                    jsmntok_t author_obj = toks[i+1];
                    for (int j = i+1; j < ntoks; j++) {
                        if (toks[j].start >= author_obj.start && toks[j].end <= author_obj.end && js_eq(json, &toks[j], "username")) {
                            author = js_get_strdup(json, &toks[j+1]); break;
                        }
                    }
                }
            }
        }
        if (content && author && channel_id) {
            printf("[%s] %s: %s\n", channel_id, author, content);
        }
        free(content); free(author); free(channel_id);
    }
    free(event);
}

int main(void) {
    const char *token = DISCORD_BOT_TOKEN;
    if (!token || !*token) { fprintf(stderr, "Set DISCORD_BOT_TOKEN environment variable (include leading 'Bot ').\n"); return 1; }

    const char *host = "gateway.discord.gg";
    const char *path = "/?v=10&encoding=json";

    runtime_t rt; memset(&rt, 0, sizeof(rt));
    rt.conn.fd = tcp_connect(host, "443");
    if (rt.conn.fd < 0) { fprintf(stderr, "TCP connect failed.\n"); return 1; }
    if (tls_handshake(&rt.conn, host) != 0) { fprintf(stderr, "TLS handshake failed.\n"); tls_close(&rt.conn); return 1; }

    char sec_key[64];
    if (ws_handshake(&rt.conn, host, path, sec_key, sizeof(sec_key)) != 0) {
        fprintf(stderr, "WebSocket handshake failed.\n"); tls_close(&rt.conn); return 1;
    }
    printf("Connected.\n");

    rt.last_seq = -1;
    rt.heartbeat_interval_ms = 0;
    rt.running = 1;

    // Reader loop: expect HELLO (op=10), then send IDENTIFY (op=2), spawn heartbeat thread
    pthread_t hb_thread;

    while (rt.running) {
        unsigned char *payload = NULL; size_t plen = 0; unsigned char opcode = 0;
        if (ws_read_frame(&rt.conn, &payload, &plen, &opcode) != 0) { fprintf(stderr, "WS read failed.\n"); break; }

        if (opcode == 0x8) { // close
            printf("Server closed connection.\n");
            free(payload);
            break;
        } else if (opcode == 0x9) { // ping -> pong
            unsigned char hdr = 0x8A; // FIN+pong
            tls_write(&rt.conn, &hdr, 1);
            free(payload);
            continue;
        } else if (opcode != 0x1) {
            // Ignore non-text frames
            free(payload);
            continue;
        }

        // Parse JSON payload
        jsmn_parser p; jsmn_init(&p);
        jsmntok_t toks[512];
        int ntoks = jsmn_parse(&p, (const char*)payload, plen, toks, 512);
        if (ntoks < 0) { free(payload); continue; }

        // Extract op, s
        int op = -1;
        int heartbeat_interval = 0;
        for (int i = 1; i < ntoks; i++) {
            if (js_eq((char*)payload, &toks[i], "op")) {
                jsmntok_t v = toks[i+1];
                if (v.type == JSMN_PRIMITIVE) op = js_get_int((char*)payload, &v);
            } else if (js_eq((char*)payload, &toks[i], "s")) {
                jsmntok_t v = toks[i+1];
                if (v.type == JSMN_PRIMITIVE) rt.last_seq = js_get_int((char*)payload, &v);
            }
        }

        if (op == 10) { // HELLO
            // Get heartbeat_interval from d.heartbeat_interval
            for (int i = 1; i < ntoks; i++) {
                if (js_eq((char*)payload, &toks[i], "d")) {
                    jsmntok_t d = toks[i+1];
                    for (int j = i+1; j < ntoks; j++) {
                        if (toks[j].start >= d.start && toks[j].end <= d.end && js_eq((char*)payload, &toks[j], "heartbeat_interval")) {
                            heartbeat_interval = js_get_int((char*)payload, &toks[j+1]);
                            break;
                        }
                    }
                    break;
                }
            }
            rt.heartbeat_interval_ms = heartbeat_interval;
            // IDENTIFY
            char *identify = json_build_identify(token);
            ws_send_text(&rt.conn, identify);
            free(identify);
            // Start heartbeat thread
            pthread_create(&hb_thread, NULL, heartbeat_thread, &rt);
        } else if (op == 0) { // DISPATCH
            handle_dispatch((char*)payload, toks, ntoks, &rt);
        }
        free(payload);
    }

    rt.running = 0;
    if (rt.heartbeat_interval_ms > 0) pthread_join(hb_thread, NULL);
    tls_close(&rt.conn);
    return 0;
}
</pre>
