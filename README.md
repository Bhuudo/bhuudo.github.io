# bhuudo.github.io
<html>
<pre>
 /*
  Browser-friendly Tiny MUD over HTTP (single file)
  - Minimal HTTP/1.1 parsing, Connection: close
  - Serves a small HTML page with a command form
  - Cookie-based sessions (SID)
  - Natural language parser (verbs, synonyms, directions, stopwords)
  - Simple dungeon: torch, key, sword, potion, goblin, hidden exit

  Build (Linux/macOS/WSL):
    gcc -O2 -Wall -o mud_http mud_http.c

  Run:
    ./mud_http 8080
    # Visit http://localhost:8080/

  Notes:
    - Single-process, single-threaded; handles one request at a time.
    - Good enough for local play and casual testing.
*/

#define _POSIX_C_SOURCE 200809L
#include <arpa/inet.h>
#include <ctype.h>
#include <errno.h>
#include <netinet/in.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>

/* -------------- Utilities -------------- */

#define MAX_BUF     65536
#define MAX_BODY    32768
#define MAX_PLAYERS 256
#define MAX_TOKENS  16

static void die(const char *fmt, ...) {
    va_list ap; va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);
    va_end(ap);
    fputc('\n', stderr);
    exit(1);
}

static int starts_with(const char *s, const char *p) {
    size_t n = strlen(p);
    return strncasecmp(s, p, n) == 0;
}

static void trim(char *s) {
    size_t n = strlen(s);
    while (n && (s[n-1]=='\r' || s[n-1]=='\n' || isspace((unsigned char)s[n-1]))) s[--n] = 0;
    size_t i=0;
    while (s[i] && isspace((unsigned char)s[i])) i++;
    if (i) memmove(s, s+i, strlen(s+i)+1);
}

static int hex(char c) {
    if (c>='0'&&c<='9') return c-'0';
    c=(char)tolower((unsigned char)c);
    if (c>='a'&&c<='f') return 10+c-'a';
    return -1;
}

static void url_decode(char *s) {
    char *o=s;
    while (*s) {
        if (*s=='+' ) { *o++=' '; s++; }
        else if (*s=='%' && isxdigit((unsigned char)s[1]) && isxdigit((unsigned char)s[2])) {
            int v = (hex(s[1])<<4) | hex(s[2]);
            *o++ = (char)v;
            s += 3;
        } else {
            *o++ = *s++;
        }
    }
    *o=0;
}

static unsigned long long rand64() {
    return ((unsigned long long)rand()<<33) ^ ((unsigned long long)rand()<<1) ^ (unsigned long long)rand();
}

/* -------------- Game data -------------- */

typedef enum { IT_TORCH, IT_KEY, IT_SWORD, IT_POTION, IT_MAX } ItemType;

typedef struct {
    const char *name;
    const char *desc;
} ItemDef;

static ItemDef ITEM_DEFS[IT_MAX] = {
    [IT_TORCH]  = {"torch",  "A stubby torch wrapped in pitch-soaked cloth."},
    [IT_KEY]    = {"key",    "A small iron key, cold to the touch."},
    [IT_SWORD]  = {"sword",  "A nicked short sword. Better than bare hands."},
    [IT_POTION] = {"potion", "A crimson potion that hums faintly."},
};

typedef struct {
    const char *name;
    const char *desc_lit;
    const char *desc_dark;
    int exits[4]; // N,E,S,W -> room index or -1
    bool locked;  // example: north locked in room 1
    bool has_hidden_west;
    bool items_present[IT_MAX];
    bool monster_present;
    int  monster_hp;
} Room;

typedef struct {
    int room;
    bool inv[IT_MAX];
    int hp;
    bool torch_lit;
    bool dead;
    bool won;
} PlayerState;

typedef struct {
    unsigned long long sid;
    time_t last_seen;
    PlayerState st;
    bool used;
} Session;

#define MAX_ROOMS 8
static Room ROOMS[MAX_ROOMS];
static Session SESS[MAX_PLAYERS];

static void init_world() {
    for (int i=0;i<MAX_ROOMS;i++) {
        ROOMS[i].name = "";
        ROOMS[i].desc_lit = ROOMS[i].desc_dark = "";
        for (int d=0; d<4; d++) ROOMS[i].exits[d] = -1;
        memset(ROOMS[i].items_present, 0, sizeof(ROOMS[i].items_present));
        ROOMS[i].locked=false; ROOMS[i].has_hidden_west=false;
        ROOMS[i].monster_present=false; ROOMS[i].monster_hp=0;
    }

    // 0 Entry
    ROOMS[0].name="Entry Hall";
    ROOMS[0].desc_lit="You stand in a stone hall. Dust swirls in your torchlight. Exits lead North and East.";
    ROOMS[0].desc_dark="Blackness presses in. You can smell damp stone. You might grope North or East.";
    ROOMS[0].exits[0]=1; ROOMS[0].exits[1]=2;
    ROOMS[0].items_present[IT_TORCH]=true;

    // 1 Gate
    ROOMS[1].name="Iron Gate";
    ROOMS[1].desc_lit="An iron portcullis bars the way North. A keyhole glints. South returns to Entry Hall.";
    ROOMS[1].desc_dark="You feel an iron lattice. The way North is blocked. South is safer.";
    ROOMS[1].exits[2]=0; ROOMS[1].exits[0]=3; ROOMS[1].locked=true;
    ROOMS[1].items_present[IT_KEY]=true;

    // 2 Armory
    ROOMS[2].name="Armory";
    ROOMS[2].desc_lit="Racks of ruined weapons. One short sword remains. A narrow West passage returns.";
    ROOMS[2].desc_dark="You bump cold metal. Something sharp scrapes your leg. West might be the way back.";
    ROOMS[2].exits[3]=0; ROOMS[2].items_present[IT_SWORD]=true;

    // 3 Gallery
    ROOMS[3].name="Gallery";
    ROOMS[3].desc_lit="Faded tapestries line the walls. A shadow stirs—a goblin snarls. A fissure West is barely visible.";
    ROOMS[3].desc_dark="A vast space. Something breathes nearby. Directions blur in darkness.";
    ROOMS[3].exits[2]=1; ROOMS[3].exits[1]=4; ROOMS[3].has_hidden_west=true;
    ROOMS[3].monster_present=true; ROOMS[3].monster_hp=8;

    // 4 Treasury
    ROOMS[4].name="Treasury";
    ROOMS[4].desc_lit="Glittering coins and a sealed vial. A single West slit leads back to the Gallery.";
    ROOMS[4].desc_dark="You slip on something round. The silence is heavy.";
    ROOMS[4].exits[3]=3; ROOMS[4].items_present[IT_POTION]=true;

    // 5 Fissure (hidden from Gallery when torch lit)
    ROOMS[5].name="Fissure";
    ROOMS[5].desc_lit="A narrow crack opens to fresh air. You could squeeze West to freedom.";
    ROOMS[5].desc_dark="Cold air brushes your face from somewhere left.";
    ROOMS[5].exits[1]=3; ROOMS[5].exits[3]=-1; // West -> outside (win)
}

/* -------------- Sessions -------------- */

static Session* get_or_create_session(unsigned long long sid_in) {
    for (int i=0;i<MAX_PLAYERS;i++) if (SESS[i].used && SESS[i].sid==sid_in) return &SESS[i];
    // create
    for (int i=0;i<MAX_PLAYERS;i++) if (!SESS[i].used) {
        SESS[i].used=true;
        SESS[i].sid = sid_in ? sid_in : rand64();
        SESS[i].last_seen=time(NULL);
        SESS[i].st.room=0; SESS[i].st.hp=10; SESS[i].st.torch_lit=false; SESS[i].st.dead=false; SESS[i].st.won=false;
        memset(SESS[i].st.inv,0,sizeof(SESS[i].st.inv));
        return &SESS[i];
    }
    return NULL;
}

/* -------------- Parser -------------- */

static const char* STOPWORDS[] = {
    "the","a","an","to","at","on","in","into","with","for","from","of","up","down",
    "please","now","that","this","my","your","our","their","his","her", NULL
};

static bool is_stop(const char *w) {
    for (int i=0; STOPWORDS[i]; i++) if (strcmp(w, STOPWORDS[i])==0) return true;
    return false;
}

static void lower(char *s){ for(;*s;s++) *s=(char)tolower((unsigned char)*s); }

typedef struct {
    char verb[16], obj1[32], prep[16], obj2[32];
    bool has_obj1, has_prep, has_obj2;
} Command;

static const char* dir_alias(const char *w){
    if(!strcmp(w,"n")||!strcmp(w,"north")) return "north";
    if(!strcmp(w,"e")||!strcmp(w,"east"))  return "east";
    if(!strcmp(w,"s")||!strcmp(w,"south")) return "south";
    if(!strcmp(w,"w")||!strcmp(w,"west"))  return "west";
    return NULL;
}
static int dir_idx(const char *d){
    if(!d) return -1;
    if(!strcmp(d,"north")) return 0;
    if(!strcmp(d,"east"))  return 1;
    if(!strcmp(d,"south")) return 2;
    if(!strcmp(d,"west"))  return 3;
    return -1;
}
static const char* verb_alias(const char *w){
    if(!strcmp(w,"go")||!strcmp(w,"move")||!strcmp(w,"walk")||!strcmp(w,"run")) return "go";
    if(!strcmp(w,"look")||!strcmp(w,"examine")||!strcmp(w,"inspect")||!strcmp(w,"l")) return "look";
    if(!strcmp(w,"take")||!strcmp(w,"grab")||!strcmp(w,"pick")) return "take";
    if(!strcmp(w,"drop")||!strcmp(w,"leave")) return "drop";
    if(!strcmp(w,"use")) return "use";
    if(!strcmp(w,"light")||!strcmp(w,"ignite")) return "light";
    if(!strcmp(w,"inventory")||!strcmp(w,"inv")||!strcmp(w,"i")) return "inventory";
    if(!strcmp(w,"attack")||!strcmp(w,"hit")||!strcmp(w,"fight")||!strcmp(w,"kill")||!strcmp(w,"strike")) return "attack";
    if(!strcmp(w,"help")||!strcmp(w,"?")) return "help";
    if(dir_alias(w)) return "go";
    return NULL;
}

static void tokenize(char *in, char *out[], int *n){
    *n=0; char *p=in;
    while (*p && *n<MAX_TOKENS){
        while (*p && isspace((unsigned char)*p)) p++;
        if(!*p) break;
        char *s=p;
        while (*p && !isspace((unsigned char)*p)) p++;
        size_t len=(size_t)(p-s);
        char *t=malloc(len+1);
        memcpy(t,s,len); t[len]=0; lower(t);
        if(!is_stop(t)) out[(*n)++]=t; else free(t);
    }
}
static void free_tokens(char *t[], int n){ for(int i=0;i<n;i++) free(t[i]); }

static void parse_command(const char *raw, Command *cmd){
    memset(cmd,0,sizeof(*cmd));
    char buf[256]; snprintf(buf,sizeof(buf),"%s", raw?raw:""); trim(buf); lower(buf);
    if(!buf[0]){ strcpy(cmd->verb,"look"); return; }

    char *tok[MAX_TOKENS]; int n=0; tokenize(buf,tok,&n);
    if(n==1){
        const char *d=dir_alias(tok[0]);
        if(d){ strcpy(cmd->verb,"go"); strcpy(cmd->obj1,d); cmd->has_obj1=true; free_tokens(tok,n); return; }
        const char *v=verb_alias(tok[0]);
        if(v){ strcpy(cmd->verb,v); free_tokens(tok,n); return; }
    }
    const char* preps[]={"on","at","with","to","into","using",NULL};
    auto bool is_prep(const char *w){ for(int i=0;preps[i];i++) if(!strcmp(w,preps[i])) return true; return false; }

    for(int i=0;i<n;i++){
        const char *w=tok[i];
        if(!cmd->verb[0]){
            const char *va=verb_alias(w);
            if(va){ strcpy(cmd->verb,va); continue; }
            const char *d=dir_alias(w);
            if(d){ strcpy(cmd->verb,"go"); strcpy(cmd->obj1,d); cmd->has_obj1=true; continue; }
            strcpy(cmd->verb,"look");
            continue;
        }
        if(!cmd->has_prep && is_prep(w)){ strcpy(cmd->prep,w); cmd->has_prep=true; continue; }
        if(!cmd->has_obj1){ strncpy(cmd->obj1,w,sizeof(cmd->obj1)-1); cmd->has_obj1=true; }
        else if(cmd->has_prep && !cmd->has_obj2){ strncpy(cmd->obj2,w,sizeof(cmd->obj2)-1); cmd->has_obj2=true; }
        else {
            // glue multi-word
            if(cmd->has_prep && cmd->has_obj2){
                size_t l=strlen(cmd->obj2); if(l<30){ strcat(cmd->obj2," "); strncat(cmd->obj2,w,30-l-1); }
            } else {
                size_t l=strlen(cmd->obj1); if(l<30){ strcat(cmd->obj1," "); strncat(cmd->obj1,w,30-l-1); }
            }
        }
    }
    free_tokens(tok,n);
}

/* -------------- Game logic -------------- */

static const char* dir_name(int i){ return (const char*[]){"North","East","South","West"}[i]; }

static void describe_room(PlayerState *ps, char *out, size_t cap){
    Room *r=&ROOMS[ps->room];
    const char *desc= ps->torch_lit? r->desc_lit : r->desc_dark;
    snprintf(out+strlen(out), cap-strlen(out), "%s: %s\n", r->name, desc);

    // exits
    snprintf(out+strlen(out), cap-strlen(out), "Exits:");
    for(int d=0; d<4; d++){
        if (d==3 && r->has_hidden_west && !ps->torch_lit) continue;
        int tgt=r->exits[d];
        if(tgt>=0){
            if(ps->room==1 && d==0 && ROOMS[1].locked)
                snprintf(out+strlen(out), cap-strlen(out), " %s(locked)", dir_name(d));
            else
                snprintf(out+strlen(out), cap-strlen(out), " %s", dir_name(d));
        }
    }
    snprintf(out+strlen(out), cap-strlen(out), "\n");
    // items
    bool any=false; for(int i=0;i<IT_MAX;i++) if(r->items_present[i]) any=true;
    if(any){
        snprintf(out+strlen(out), cap-strlen(out), "You see:\n");
        for(int i=0;i<IT_MAX;i++) if(r->items_present[i])
            snprintf(out+strlen(out), cap-strlen(out), " - %s: %s\n", ITEM_DEFS[i].name, ITEM_DEFS[i].desc);
    }
    if(r->monster_present)
        snprintf(out+strlen(out), cap-strlen(out), "An angry goblin glares at you. HP: %d\n", r->monster_hp);
}

static int item_from(const char *name){
    if(!name||!*name) return -1;
    for(int i=0;i<IT_MAX;i++){
        if (strstr(ITEM_DEFS[i].name,name)==ITEM_DEFS[i].name || strstr(name,ITEM_DEFS[i].name)) return i;
    }
    if(!strcmp(name,"blade")) return IT_SWORD;
    if(!strcmp(name,"lamp")||!strcmp(name,"light")) return IT_TORCH;
    if(!strcmp(name,"vial")) return IT_POTION;
    return -1;
}

static void do_look(PlayerState *ps,char *o,size_t c){ describe_room(ps,o,c); }
static void do_inv(PlayerState *ps,char *o,size_t c){
    bool any=false; for(int i=0;i<IT_MAX;i++) if(ps->inv[i]) any=true;
    if(!any){ snprintf(o+strlen(o), c-strlen(o), "Your inventory is empty.\n"); return; }
    snprintf(o+strlen(o), c-strlen(o), "You carry:\n");
    for(int i=0;i<IT_MAX;i++) if(ps->inv[i]) snprintf(o+strlen(o), c-strlen(o), " - %s\n", ITEM_DEFS[i].name);
}
static void do_go(PlayerState *ps, const char *dir, char *o, size_t c){
    if(!dir){ snprintf(o+strlen(o), c-strlen(o), "Go where?\n"); return; }
    int d=dir_idx(dir); if(d<0){ snprintf(o+strlen(o), c-strlen(o), "I don't recognize that direction.\n"); return; }
    Room *r=&ROOMS[ps->room];
    if(d==3 && r->has_hidden_west && !ps->torch_lit){ snprintf(o+strlen(o), c-strlen(o), "You feel a wall where 'West' should be.\n"); return; }
    if(ps->room==1 && d==0 && ROOMS[1].locked){ snprintf(o+strlen(o), c-strlen(o), "The iron gate blocks the way North.\n"); return; }
    if(ps->room==5 && d==3){ ps->won=true; snprintf(o+strlen(o), c-strlen(o), "You squeeze through the fissure and emerge under open sky. You win!\n"); return; }
    int tgt=r->exits[d]; if(tgt<0){ snprintf(o+strlen(o), c-strlen(o), "No passage that way.\n"); return; }
    ps->room=tgt; do_look(ps,o,c);
}
static void do_take(PlayerState *ps, const char *obj, char *o, size_t c){
    int it=item_from(obj);
    if(it<0){ snprintf(o+strlen(o), c-strlen(o), "Take what?\n"); return; }
    Room *r=&ROOMS[ps->room];
    if(!r->items_present[it]){ snprintf(o+strlen(o), c-strlen(o), "There is no %s here.\n", ITEM_DEFS[it].name); return; }
    int cnt=0; for(int i=0;i<IT_MAX;i++) if(ps->inv[i]) cnt++;
    if(cnt>=8){ snprintf(o+strlen(o), c-strlen(o), "You're carrying too much.\n"); return; }
    ps->inv[it]=true; r->items_present[it]=false;
    snprintf(o+strlen(o), c-strlen(o), "You take the %s.\n", ITEM_DEFS[it].name);
}
static void do_drop(PlayerState *ps, const char *obj, char *o, size_t c){
    int it=item_from(obj);
    if(it<0){ snprintf(o+strlen(o), c-strlen(o), "Drop what?\n"); return; }
    if(!ps->inv[it]){ snprintf(o+strlen(o), c-strlen(o), "You don't have a %s.\n", ITEM_DEFS[it].name); return; }
    ps->inv[it]=false; ROOMS[ps->room].items_present[it]=true;
    if(it==IT_TORCH) ps->torch_lit=false;
    snprintf(o+strlen(o), c-strlen(o), "You drop the %s.\n", ITEM_DEFS[it].name);
}
static void do_light(PlayerState *ps, const char *obj, char *o, size_t c){
    if(!obj||!*obj) obj="torch";
    int it=item_from(obj);
    if(it!=IT_TORCH || !ps->inv[IT_TORCH]){ snprintf(o+strlen(o), c-strlen(o), "You fumble for a torch you don't possess.\n"); return; }
    if(ps->torch_lit){ snprintf(o+strlen(o), c-strlen(o), "Your torch already burns.\n"); return; }
    ps->torch_lit=true; snprintf(o+strlen(o), c-strlen(o), "You spark the torch. Warm light floods the dark.\n");
}
static void goblin_counter(PlayerState *ps, char *o, size_t c){
    Room *r=&ROOMS[ps->room]; if(!(r->monster_present && r->monster_hp>0)) return;
    int dmg=(rand()%3==0)?2:1; ps->hp-=dmg;
    snprintf(o+strlen(o), c-strlen(o), "The goblin slashes you for %d damage (HP %d).\n", dmg, ps->hp);
    if(ps->hp<=0){ ps->dead=true; snprintf(o+strlen(o), c-strlen(o), "You collapse. Darkness takes you. Game over.\n"); }
}
static void do_use(PlayerState *ps, const char *obj1, const char *prep, const char *obj2, char *o, size_t c){
    (void)prep; (void)obj2;
    if(!obj1){ snprintf(o+strlen(o), c-strlen(o), "Use what?\n"); return; }
    int it=item_from(obj1);
    if(it<0 || !ps->inv[it]){ snprintf(o+strlen(o), c-strlen(o), "You don't have that.\n"); return; }
    if(it==IT_KEY && ps->room==1){
        if(!ROOMS[1].locked){ snprintf(o+strlen(o), c-strlen(o), "The gate is already unlocked.\n"); return; }
        ROOMS[1].locked=false; snprintf(o+strlen(o), c-strlen(o), "You turn the key. With a rasp, the gate unlocks.\n"); return;
    }
    if(it==IT_POTION){
        ps->hp+=5; if(ps->hp>12) ps->hp=12; ps->inv[IT_POTION]=false;
        snprintf(o+strlen(o), c-strlen(o), "You drink the potion. Vitality surges (HP now %d).\n", ps->hp); return;
    }
    if(it==IT_TORCH && !ps->torch_lit){ do_light(ps,"torch",o,c); return; }
    snprintf(o+strlen(o), c-strlen(o), "Nothing happens.\n");
}
static void do_attack(PlayerState *ps, const char *obj, char *o, size_t c){
    Room *r=&ROOMS[ps->room];
    if(!r->monster_present || r->monster_hp<=0){ snprintf(o+strlen(o), c-strlen(o), "There's nothing here to attack.\n"); return; }
    int base= ps->inv[IT_SWORD] ? 4 : 2;
    int dealt = base + (rand()%3 - 1);
    if(dealt<1) dealt=1;
    r->monster_hp -= dealt;
    snprintf(o+strlen(o), c-strlen(o), "You strike the goblin for %d damage. ", dealt);
    if(r->monster_hp<=0){ r->monster_present=false; snprintf(o+strlen(o), c-strlen(o), "It falls with a gasp.\n"); }
    else { snprintf(o+strlen(o), c-strlen(o), "It snarls (HP %d).\n", r->monster_hp); goblin_counter(ps,o,c); }
}
static void do_help(char *o, size_t c){
    snprintf(o+strlen(o), c-strlen(o),
        "Commands:\n"
        " - look / l\n"
        " - go <north|east|south|west> (or n/e/s/w)\n"
        " - take <item>, drop <item>\n"
        " - light torch, use <item> [on <target>]\n"
        " - attack [goblin]\n"
        " - inventory / i\n"
        "Goal: light the torch, unlock the gate, survive the goblin, and find a way out.\n"
    );
}
static void run_command(PlayerState *ps, Command *cmd, char *o, size_t c){
    if(ps->dead){ snprintf(o+strlen(o), c-strlen(o), "You are dead. Refresh /reset to start anew.\n"); return; }
    if(ps->won){ snprintf(o+strlen(o), c-strlen(o), "You already escaped. Refresh /reset to play again.\n"); return; }

    if(!strcmp(cmd->verb,"look")) do_look(ps,o,c);
    else if(!strcmp(cmd->verb,"inventory")) do_inv(ps,o,c);
    else if(!strcmp(cmd->verb,"go")){
        const char *dir=NULL;
        if(cmd->has_obj1){
            if(!strcmp(cmd->obj1,"gate") && ps->room==1) dir="north";
            else dir=dir_alias(cmd->obj1);
        }
        if(!dir && cmd->has_prep && cmd->has_obj2) dir=dir_alias(cmd->obj2);
        do_go(ps,dir,o,c);
    } else if(!strcmp(cmd->verb,"take")) do_take(ps, cmd->has_obj1?cmd->obj1:NULL, o,c);
    else if(!strcmp(cmd->verb,"drop")) do_drop(ps, cmd->has_obj1?cmd->obj1:NULL, o,c);
    else if(!strcmp(cmd->verb,"light")) do_light(ps, cmd->has_obj1?cmd->obj1:"torch", o,c);
    else if(!strcmp(cmd->verb,"use")) do_use(ps, cmd->has_obj1?cmd->obj1:NULL, cmd->has_prep?cmd->prep:NULL, cmd->has_obj2?cmd->obj2:NULL, o,c);
    else if(!strcmp(cmd->verb,"attack")) do_attack(ps, cmd->has_obj1?cmd->obj1:NULL, o,c);
    else if(!strcmp(cmd->verb,"help")) do_help(o,c);
    else snprintf(o+strlen(o), c-strlen(o), "You mutter '%s' to no effect. Try 'help'.\n", cmd->verb);

    // Reveal hidden west exit when torch lit in Gallery
    if(ps->room==3 && ps->torch_lit) ROOMS[3].exits[3]=5;
}

/* -------------- HTTP -------------- */

typedef struct {
    char method[8];
    char path[1024];
    char query[2048];
    char cookie[2048];
    char content_type[128];
    int  content_length;
    char body[MAX_BODY];
} HttpRequest;

static void send_all(int fd, const char *buf, size_t n){
    size_t off=0; while(off<n){ ssize_t w=send(fd, buf+off, n-off, 0); if(w<=0) return; off+= (size_t)w; }
}

static void http_respond_html(int fd, unsigned long long sid, const char *html){
    char hdr[1024];
    int n = snprintf(hdr, sizeof(hdr),
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: text/html; charset=utf-8\r\n"
        "Cache-Control: no-store\r\n"
        "Set-Cookie: SID=%llu; Path=/; HttpOnly\r\n"
        "Content-Length: %zu\r\n"
        "Connection: close\r\n"
        "\r\n", sid, strlen(html));
    send_all(fd, hdr, (size_t)n);
    send_all(fd, html, strlen(html));
}

static void http_respond_400(int fd, const char *msg){
    char body[512];
    snprintf(body,sizeof(body),
        "<!doctype html><html><body><pre>Bad Request: %s</pre></body></html>", msg);
    char hdr[512];
    int n=snprintf(hdr,sizeof(hdr),
        "HTTP/1.1 400 Bad Request\r\n"
        "Content-Type: text/html; charset=utf-8\r\n"
        "Content-Length: %zu\r\n"
        "Connection: close\r\n\r\n", strlen(body));
    send_all(fd,hdr,(size_t)n);
    send_all(fd,body,strlen(body));
}

static void parse_request(int fd, HttpRequest *req){
    memset(req,0,sizeof(*req));
    char buf[MAX_BUF]; int rcv = recv(fd, buf, sizeof(buf)-1, 0);
    if(rcv<=0) die("recv failed: %s", strerror(errno));
    buf[rcv]=0;

    // Find end of headers
    char *hdr_end = NULL;
    for(int i=0; i<rcv-3; i++){
        if(buf[i]=='\r' && buf[i+1]=='\n' && buf[i+2]=='\r' && buf[i+3]=='\n'){ hdr_end = buf+i+4; break; }
        if(buf[i]=='\n' && buf[i+1]=='\n'){ hdr_end = buf+i+2; break; }
    }
    if(!hdr_end) die("incomplete headers");

    // Request line
    char *p=buf;
    char *line_end = strstr(p,"\r\n"); if(!line_end) line_end=strstr(p,"\n");
    if(!line_end) die("no request line");
    *line_end=0;
    {
        char httpver[32];
        if(sscanf(p,"%7s %1023s %31s", req->method, req->path, httpver)!=3) die("bad request line");
    }

    // Headers
    p=line_end+1;
    while(p < hdr_end && *p){
        char *e = strstr(p,"\r\n"); if(!e) e=strstr(p,"\n");
        if(!e) break;
        *e=0; char *line=p; trim(line);
        if(*line==0) break;
        if(starts_with(line,"Cookie:")){
            const char *v=line+7; while(*v && isspace((unsigned char)*v)) v++;
            snprintf(req->cookie,sizeof(req->cookie),"%s", v);
        } else if(starts_with(line,"Content-Type:")){
            const char *v=line+13; while(*v && isspace((unsigned char)*v)) v++;
            snprintf(req->content_type,sizeof(req->content_type),"%127s", v);
        } else if(starts_with(line,"Content-Length:")){
            const char *v=line+15; while(*v && isspace((unsigned char)*v)) v++;
            req->content_length = atoi(v);
        }
        p = e+1;
    }

    // Query
    {
        char *q=strchr(req->path,'?');
        if(q){ *q=0; snprintf(req->query,sizeof(req->query),"%s", q+1); }
    }

    // Body
    int header_bytes = (int)(hdr_end - buf);
    int inbuf_body = rcv - header_bytes;
    if(req->content_length>0){
        int need = req->content_length;
        if(inbuf_body > need) inbuf_body = need;
        memcpy(req->body, hdr_end, inbuf_body);
        int remaining = need - inbuf_body;
        int off = inbuf_body;
        while(remaining>0 && off < (int)sizeof(req->body)-1){
            int rr = recv(fd, req->body+off, remaining, 0);
            if(rr<=0) break;
            off += rr; remaining -= rr;
        }
        req->body[ (off < (int)sizeof(req->body)-1) ? off : (int)sizeof(req->body)-1 ] = 0;
    }
}

static unsigned long long parse_sid_cookie(const char *cookie){
    if(!cookie||!*cookie) return 0ULL;
    const char *p=strstr(cookie,"SID=");
    if(!p) return 0ULL;
    p+=4;
    char num[32]={0}; int i=0;
    while(*p && *p!=';' && !isspace((unsigned char)*p) && i<(int)sizeof(num)-1) num[i++]=*p++;
    return strtoull(num,NULL,10);
}

static void parse_cmd_from_form(const char *s, char *out, size_t cap){
    out[0]=0; if(!s) return;
    // supports cmd=... optionally among other k=v pairs
    const char *p=strstr(s,"cmd=");
    if(!p) return;
    p+=4;
    char val[2048]; int i=0;
    while(*p && *p!='&' && i<(int)sizeof(val)-1) val[i++]=*p++;
    val[i]=0; url_decode(val);
    snprintf(out,cap,"%s",val);
}

/* -------------- HTML rendering -------------- */

static void escape_html(const char *src, char *dst, size_t cap){
    size_t off=0;
    for(; *src && off+6<cap; src++){
        char ch=*src;
        if(ch=='&'){ off+=snprintf(dst+off,cap-off,"&amp;"); }
        else if(ch=='<'){ off+=snprintf(dst+off,cap-off,"&lt;"); }
        else if(ch=='>'){ off+=snprintf(dst+off,cap-off,"&gt;"); }
        else if(ch=='"'){ off+=snprintf(dst+off,cap-off,"&quot;"); }
        else { dst[off++]=ch; }
    }
    dst[off]=0;
}

static void build_page(const char *gameout, PlayerState *ps, unsigned long long sid, char *html, size_t cap){
    char esc[32768]; escape_html(gameout, esc, sizeof(esc));
    snprintf(html, cap,
        "<!doctype html><html><head>"
        "<meta charset=\"utf-8\">"
        "<meta name=\"viewport\" content=\"width=device-width,initial-scale=1\">"
        "<title>Tiny HTTP MUD</title>"
        "<style>"
        "body{font-family:ui-monospace,monospace;background:#0b0d10;color:#cde;}"
        "main{max-width:900px;margin:2rem auto;padding:1rem;border:1px solid #234;background:#11161c;border-radius:8px;}"
        "pre{white-space:pre-wrap;background:#0e1217;padding:1rem;border-radius:6px;min-height:16rem;}"
        "form{margin-top:1rem;display:flex;gap:.5rem}"
        "input[type=text]{flex:1;padding:.6rem;border:1px solid #355;background:#0d131a;color:#def;border-radius:4px}"
        "button{padding:.6rem 1rem;border:1px solid #46a;background:#17343f;color:#e8ffff;border-radius:4px;cursor:pointer}"
        "small{color:#8aa}"
        "</style></head><body><main>"
        "<h1>Tiny HTTP MUD</h1>"
        "<pre>%s\n\nHP: %d | Torch: %s | Room: %s\n"
        "Try: look, take torch, light torch, e, take sword, w, n, take key, use key, n, attack goblin, w.</pre>"
        "<form method=\"POST\" action=\"/\">"
        "<input name=\"cmd\" type=\"text\" autofocus placeholder=\"Enter command...\"/>"
        "<button type=\"submit\">Send</button>"
        "</form>"
        "<p><small>Session: %llu · <a href=\"/reset\" style=\"color:#8fd\">Reset</a></small></p>"
        "</main></body></html>",
        esc, ps->hp, ps->torch_lit?"lit":"dark", ROOMS[ps->room].name, sid
    );
}

/* -------------- Main -------------- */

static void format_prompt(PlayerState *ps, char *o, size_t c){
    snprintf(o+strlen(o), c-strlen(o), "\n");
}

int main(int argc, char **argv){
    srand((unsigned)time(NULL));
    init_world();

    int port = (argc>1)? atoi(argv[1]) : 8080;

    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if(listen_fd<0) die("socket: %s", strerror(errno));
    int opt=1; setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr; memset(&addr,0,sizeof(addr));
    addr.sin_family=AF_INET; addr.sin_port=htons((uint16_t)port); addr.sin_addr.s_addr=htonl(INADDR_ANY);
    if(bind(listen_fd,(struct sockaddr*)&addr,sizeof(addr))<0) die("bind: %s", strerror(errno));
    if(listen(listen_fd, 16)<0) die("listen: %s", strerror(errno));

    fprintf(stderr,"MUD server listening on http://localhost:%d/\n", port);

    for(;;){
        struct sockaddr_in cli; socklen_t clen=sizeof(cli);
        int fd = accept(listen_fd, (struct sockaddr*)&cli, &clen);
        if(fd<0) continue;

        HttpRequest req; parse_request(fd,&req);

        unsigned long long sid = parse_sid_cookie(req.cookie);
        Session *sess = get_or_create_session(sid);
        if(!sess){ http_respond_400(fd,"Server full"); close(fd); continue; }
        sid = sess->sid; sess->last_seen=time(NULL);

        char gameout[32768]; gameout[0]=0;

        // Paths
        if(!strcmp(req.path,"/reset")){
            // Reset player and world (simple local instance)
            init_world();
            sess->st.room=0; sess->st.hp=10; sess->st.torch_lit=false; sess->st.dead=false; sess->st.won=false;
            memset(sess->st.inv,0,sizeof(sess->st.inv));
            snprintf(gameout, sizeof(gameout), "You gather yourself at the Entry Hall once more.\n");
        } else if(!strcmp(req.path,"/")){
            // Command handling
            char cmdline[2048]={0};
            if(req.query[0]){ parse_cmd_from_form(req.query, cmdline, sizeof(cmdline)); }
            if(!cmdline[0] && !strcmp(req.method,"POST") && req.content_length>0){
                // Expect application/x-www-form-urlencoded
                parse_cmd_from_form(req.body, cmdline, sizeof(cmdline));
            }

            if(cmdline[0]){
                Command c; parse_command(cmdline, &c);
                run_command(&sess->st, &c, gameout, sizeof(gameout));
                format_prompt(&sess->st, gameout, sizeof(gameout));
            } else {
                // First visit: show look
                Command c; parse_command("look",&c);
                run_command(&sess->st, &c, gameout, sizeof(gameout));
                format_prompt(&sess->st, gameout, sizeof(gameout));
            }
        } else {
            http_respond_400(fd,"Unknown path"); close(fd); continue;
        }

        // Render HTML page
        char html[65536];
        build_page(gameout, &sess->st, sid, html, sizeof(html));
        http_respond_html(fd, sid, html);
        close(fd);
    }
    return 0;
} 
</pre> 
</html>
