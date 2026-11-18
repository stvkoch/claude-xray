#!/usr/bin/env python3
"""claude-xray: live X-ray of a Claude Code session.

Reads the session transcript Claude Code writes to ~/.claude/projects/ and
shows, refreshing every second:
  - context size of the main conversation (and peak, compactions)
  - subagents: launched / running / finished, with their own context size
      focus mode (f): follow one subagent live, tail -f style
      brief vs result (b): what the parent asked, what came back, and whether it was used
  - skills: invoked (Skill tool or /command), available count, newly added
  - a feed of the latest tool calls (main thread and subagents)

Read-only: it never writes to Claude's files.

Usage:
  claude-xray                 # pick a session from a list (Textual UI)
  claude-xray --latest        # follow the newest session, switching after /clear
  claude-xray --all           # follow the newest session in any project
  claude-xray --session ID    # follow a specific session id
  claude-xray --once          # print one snapshot and exit
  claude-xray --limit 1000000 # context window size used for the % bar (default: from the model)
"""
import argparse, hashlib, json, os, re, shutil, sys, time
from datetime import datetime
from pathlib import Path

PROJECTS = Path.home() / ".claude" / "projects"
NO_COLOR = not sys.stdout.isatty() or os.environ.get("NO_COLOR")

def c(code, s):
    return s if NO_COLOR else f"\033[{code}m{s}\033[0m"
BOLD, DIM, RED, GREEN, YELLOW, CYAN, MAGENTA = "1", "2", "31", "32", "33", "36", "35"

def fmt_tok(n):
    if n >= 1_000_000: return f"{n/1_000_000:.2f}M"
    if n >= 1000: return f"{n/1000:.1f}k"
    return str(n)

def hhmm(ts):
    try:
        return datetime.fromisoformat(ts.replace("Z", "+00:00")).astimezone().strftime("%H:%M:%S")
    except Exception:
        return "--:--:--"

def project_dir_for(cwd):
    return PROJECTS / re.sub(r"[/.]", "-", str(cwd))

_ENTRYPOINTS = {}


def is_headless(f):
    """True for sessions started by `claude -p` / the SDK (entrypoint "sdk-cli"), not typed in a terminal."""
    if f not in _ENTRYPOINTS:
        ep = None
        try:
            with open(f, encoding="utf-8", errors="replace") as fh:
                for i, l in enumerate(fh):
                    if i > 50 or ep:
                        break
                    if '"entrypoint"' in l:
                        try:
                            ep = json.loads(l).get("entrypoint")
                        except Exception:
                            pass
        except OSError:
            pass
        if ep is None:  # not written yet: decide later
            return False
        _ENTRYPOINTS[f] = ep.startswith("sdk")
    return _ENTRYPOINTS[f]


def session_files(all_projects, cwd, headless=False):
    dirs = [d for d in PROJECTS.iterdir() if d.is_dir()] if all_projects else [project_dir_for(cwd)]
    files = (f for d in dirs if d.exists() for f in d.glob("*.jsonl"))
    return [f for f in files if headless or not is_headless(f)]


def newest_session(all_projects, cwd, headless=False):
    files = session_files(all_projects, cwd, headless)
    return max(files, key=lambda f: f.stat().st_mtime) if files else None

def born(f):
    st = f.stat()
    return getattr(st, "st_birthtime", st.st_ctime)

def context_of(usage):
    return (usage.get("input_tokens", 0) + usage.get("cache_read_input_tokens", 0)
            + usage.get("cache_creation_input_tokens", 0))

def short_input(name, inp):
    if name == "Bash": return inp.get("description") or inp.get("command", "")[:60]
    if name in ("Read", "Write", "Edit"): return Path(inp.get("file_path", "")).name
    if name == "Agent": return inp.get("description", "")
    if name == "Skill": return inp.get("skill", "")
    if name in ("Grep", "Glob"): return inp.get("pattern", "")
    return next((str(v)[:50] for v in inp.values() if isinstance(v, str)), "")


def result_text(content, cap=20_000):
    if isinstance(content, list):
        content = "\n".join(b.get("text", "") if b.get("type") == "text" else f"[{b.get('type')}]"
                            for b in content if isinstance(b, dict))
    text = str(content or "")
    return text if len(text) <= cap else text[:cap] + f"\n… (+{len(text) - cap} chars)"


def find_skill_md(name, cwd):
    base = name.split(":")[-1]
    roots = [Path(cwd) / ".claude" / "skills"] if cwd else []
    roots += [Path.home() / ".agents" / "skills", Path.home() / ".claude" / "skills"]
    for r in roots:
        if (r / base / "SKILL.md").exists():
            return r / base / "SKILL.md"
    plug = Path.home() / ".claude" / "plugins"
    return next(plug.glob(f"**/skills/{base}/SKILL.md"), None) if plug.exists() else None


class Tail:
    """Reads only the new complete lines of a growing JSONL file."""
    def __init__(self, path):
        self.path, self.pos, self.buf = Path(path), 0, ""

    def read(self):
        try:
            with open(self.path, "r", encoding="utf-8", errors="replace") as f:
                f.seek(self.pos)
                data = f.read()
                self.pos = f.tell()
        except FileNotFoundError:
            return []
        self.buf += data
        *lines, self.buf = self.buf.split("\n")
        out = []
        for l in lines:
            try: out.append(json.loads(l))
            except Exception: pass
        return out


def msg_hash(msg):
    return hashlib.sha256(json.dumps(msg, sort_keys=True, separators=(",", ":"), ensure_ascii=False)
                          .encode()).hexdigest()


# Models with a 1M-token window (Claude API model reference, 2026-11). Everything else,
# e.g. Haiku 4.5, Sonnet 4.5, Opus 4.5 and older, has 200K unless run with the [1m] suffix.
ONE_M_MODELS = re.compile(r"claude-(fable|mythos)|claude-(opus|sonnet|haiku)-5|claude-opus-4-[678]|claude-sonnet-4-6")


def context_window(model, peak=0):
    """Context window for a model id; past-200K usage proves a 1M window whatever the id says."""
    if "[1m]" in (model or "").lower() or ONE_M_MODELS.search(model or "") or peak > 200_000:
        return 1_000_000
    return 200_000


def is_real_prompt(text):
    """A message typed by the user, not a tag-wrapped injection or a subagent hand-back."""
    return bool(text) and not text.startswith("<") and not text.startswith("Another Claude session sent a message")


def first_text(content):
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        return next((b.get("text", "") for b in content if isinstance(b, dict) and b.get("type") == "text"), "")
    return ""


class Agent:
    def __init__(self, aid):
        self.id, self.desc, self.type, self.model = aid, "", "", ""
        self.finished, self.context, self.tools, self.last_tool = False, 0, 0, ""
        self.skills, self.start, self.end = [], "", ""
        self.tool_use_id = ""   # the parent's Agent tool_use id (from meta.json)
        self.brief = ""         # the prompt the parent wrote
        self.report = ""        # SubagentHandback message, else the latest text the agent wrote
        self.handed_back = False
        self.events = []        # (ts, kind, data) in arrival order, for the focus view
        self.tool_names = {}    # tool_use_id -> tool name


class Session:
    def __init__(self, path):
        self.path = Path(path)
        self.id = self.path.stem
        self.tail = Tail(path)
        self.sub_dir = self.path.parent / self.id / "subagents"
        self.agent_tails, self.agents = {}, {}
        self.context, self.peak, self.out_tokens, self.compactions = 0, 0, 0, 0
        self.turns, self.model = 0, ""
        self.skill_calls = []      # (time, who, skill, how)
        self.skills_available = 0
        self.skills_added = []     # names that appeared in a later skill listing
        self._known_skills = None
        self.skill_reads = {}      # skill name -> set of files read inside it
        self.feed = []             # (time, who, tool, detail)
        self.agent_calls = 0
        self.last_prompt = ""
        self.results = {}          # tool_use_id -> (time, text, is_error)
        self.ctx_hist = []         # (time, ctx, input, cache_read, cache_new, output, prompt, msg_id)
        self.compact_ts = []
        self.cwd = ""
        self.main_out = []         # (time, text) the main thread wrote: replies and tool inputs
        self.prompt_ts = []        # times of real user prompts
        self.handback = {}         # agent id -> time its report reached the main thread
        # exact API requests, as Claude Code records them (see request_context)
        self.blobs = {}            # message hash -> message
        self.shapes = {}           # shape hash -> {"system": [...], "tools": [...]}
        self.requests = []         # api-request entries, in order
        self.req_of_msg = {}       # assistant message id -> api-request id that produced it
        self._run = []             # transcript entries of the API message being assembled

    # ---- main transcript -------------------------------------------------
    def update(self):
        for e in self.tail.read():
            self._event(e, who="main")
        if self.sub_dir.exists():
            for f in self.sub_dir.glob("agent-*.jsonl"):
                aid = f.stem.replace("agent-", "")
                t = self.agent_tails.setdefault(aid, Tail(f))
                a = self.agents.setdefault(aid, Agent(aid))
                for e in t.read():
                    self._agent_event(a, e)
            for m in self.sub_dir.glob("agent-*.meta.json"):
                aid = m.name.replace("agent-", "").replace(".meta.json", "")
                a = self.agents.setdefault(aid, Agent(aid))
                try:
                    meta = json.loads(m.read_text())
                    a.desc = meta.get("description", a.desc)
                    a.type = meta.get("agentType", a.type)
                    a.model = meta.get("model", a.model)
                    a.tool_use_id = meta.get("toolUseId", a.tool_use_id)
                    a.finished = bool(meta.get("finished", False))
                except Exception:
                    pass

    def _event(self, e, who):
        t = e.get("type")
        ts = e.get("timestamp", "")
        self.cwd = self.cwd or e.get("cwd", "")
        if t in ("user", "assistant") and not e.get("isSidechain"):
            self._remember(e)
        if t == "api-request-blob":
            self.blobs[e.get("hash")] = e.get("message")
        elif t == "api-request-shape":
            self.shapes[e.get("shapeHash")] = e.get("shape") or {}
        elif t == "api-request":
            self._flush_run()
            self.requests.append(e)
        if t == "assistant" and not e.get("isSidechain"):
            m = e.get("message", {})
            if m.get("id") and e.get("requestRef"):
                self.req_of_msg[m["id"]] = e["requestRef"]
            self.model = m.get("model", self.model)
            u = m.get("usage")
            if u:
                self.context = context_of(u)
                self.peak = max(self.peak, self.context)
                mid = m.get("id")
                rec = (ts, self.context, u.get("input_tokens", 0), u.get("cache_read_input_tokens", 0),
                       u.get("cache_creation_input_tokens", 0), u.get("output_tokens", 0), self.last_prompt, mid)
                if self.ctx_hist and mid and self.ctx_hist[-1][-1] == mid:
                    self.out_tokens += rec[5] - self.ctx_hist[-1][5]
                    self.ctx_hist[-1] = rec
                else:
                    self.out_tokens += rec[5]
                    self.ctx_hist.append(rec)
            for blk in m.get("content", []):
                if blk.get("type") == "tool_use":
                    self._tool(ts, who, blk["name"], blk.get("input", {}), blk.get("id"))
                    self.main_out.append((ts, json.dumps(blk.get("input", {}), ensure_ascii=False)[:20_000]))
                elif blk.get("type") == "text" and blk.get("text"):
                    self.main_out.append((ts, blk["text"][:20_000]))
        elif t == "user":
            content = e.get("message", {}).get("content")
            self._results(ts, content)
            text = first_text(content)
            for aid in re.findall(r'<agent-message from="(\w+)"|<task-id>(\w+)</task-id>', text):
                self.handback.setdefault(aid[0] or aid[1], ts)
            if isinstance(content, str):
                cmd = re.search(r"<command-name>/?([^<]+)</command-name>", content)
                if cmd:
                    args = re.search(r"<command-args>(.*?)</command-args>", content, re.S)
                    self.skill_calls.append((ts, who, cmd.group(1).strip(), "/command", None,
                                             args.group(1).strip() if args else ""))
                elif is_real_prompt(content):
                    self.turns += 1
                    self.prompt_ts.append(ts)
                    self.last_prompt = content.strip().replace("\n", " ")
            elif isinstance(content, list) and any(b.get("type") == "text" and is_real_prompt(b.get("text", ""))
                                                   for b in content if isinstance(b, dict)):
                self.turns += 1
                self.prompt_ts.append(ts)
        elif t == "attachment":
            a = e.get("attachment", {})
            if a.get("type") == "skill_listing":
                names = [n.rstrip(":") for n in re.findall(r"^- ([\w:.\-/]+)", a.get("content", ""), re.M)]
                if self._known_skills is not None:
                    for n in names:
                        if n not in self._known_skills:
                            self.skills_added.append((ts, n))
                self._known_skills = (self._known_skills or set()) | set(names)
                self.skills_available = len(self._known_skills)
        elif t == "system" and e.get("subtype") in ("compact_boundary", "compaction"):
            self.compactions += 1
            self.compact_ts.append(ts)

    # ---- request reconstruction -------------------------------------------
    # Claude Code logs each API request as: a shape (system prompt + tools, by hash), and
    # messages = previous request's first `keep` messages + the `tail` hashes. A message that
    # is already a transcript entry isn't stored again as a blob, so we hash transcript
    # messages too, the same way: sha256 of sorted compact JSON of {role, content}. One API
    # message can span several transcript entries (one per block), so runs are merged as well.
    def _remember(self, e):
        m = e.get("message", {})
        msg = {"role": m.get("role"), "content": m.get("content")}
        self.blobs.setdefault(msg_hash(msg), msg)
        if isinstance(msg["content"], str):  # sometimes sent as a single text block
            wrapped = {"role": msg["role"], "content": [{"type": "text", "text": msg["content"]}]}
            self.blobs.setdefault(msg_hash(wrapped), wrapped)
        last = self._run[-1] if self._run else None
        if last and (last[0] != msg["role"] or (msg["role"] == "assistant" and last[1] != m.get("id"))):
            self._flush_run()
        self._run.append((msg["role"], m.get("id"), msg["content"]))

    def _flush_run(self):
        if len(self._run) > 1:
            content = []
            for _, _, c in self._run:
                content += c if isinstance(c, list) else [{"type": "text", "text": c}]
            msg = {"role": self._run[0][0], "content": content}
            self.blobs.setdefault(msg_hash(msg), msg)
        self._run = []

    def request_context(self, req):
        """(shape, messages) exactly as sent with this api-request. Unresolved messages are None."""
        msgs = []
        src = req.get("querySource")
        for r in self.requests:
            if r.get("querySource") != src:
                continue
            msgs = msgs[:int(r.get("keep") or 0)] + [self.blobs.get(h) for h in r.get("tail", [])]
            if r is req:
                break
        return self.shapes.get(req.get("shapeHash"), {}), msgs

    def request_for(self, msg_id=None):
        """The request behind an assistant message, or the latest main-thread request."""
        if msg_id:
            rid = self.req_of_msg.get(msg_id)
            return next((r for r in self.requests if r.get("id") == rid), None)
        return self.requests[-1] if self.requests else None

    def real_context(self, req):
        """The context size the API reported for the response to this request, if known."""
        rid = req.get("id")
        for rec in reversed(self.ctx_hist):
            if self.req_of_msg.get(rec[-1]) == rid:
                return rec[1]
        return 0

    def _results(self, ts, content):
        if not isinstance(content, list):
            return
        for b in content:
            if isinstance(b, dict) and b.get("type") == "tool_result":
                self.results[b.get("tool_use_id")] = (ts, result_text(b.get("content")), bool(b.get("is_error")))

    def _tool(self, ts, who, name, inp, tid=None):
        self.feed.append((ts, who, name, short_input(name, inp), tid, inp))
        self.feed = sorted(self.feed, key=lambda x: x[0])[-300:]
        if name in ("Agent", "Task"):
            self.agent_calls += 1
        if name == "Skill":
            self.skill_calls.append((ts, who, inp.get("skill", "?"), "auto/Skill", tid, inp.get("args", "")))
        text = inp.get("file_path", "") if name == "Read" else inp.get("command", "") if name == "Bash" else ""
        for m in re.finditer(r"/skills/(?:[^/\s]+/)*?([^/\s]+)/(SKILL\.md|(?:references|scripts|assets)/[^\s'\"|;&]+)", text):
            self.skill_reads.setdefault(m.group(1), set()).add(Path(m.group(2)).name)

    # ---- subagent transcripts -------------------------------------------
    def _agent_event(self, a, e):
        t, ts = e.get("type"), e.get("timestamp", "")
        if t not in ("user", "assistant"):
            return
        a.start = a.start or ts
        a.end = ts or a.end
        content = e.get("message", {}).get("content")
        if t == "user":
            self._results(ts, content)
            if isinstance(content, str):
                if not a.brief and not a.events:
                    a.brief = content
                    a.events.append((ts, "brief", content))
                else:
                    a.events.append((ts, "note", content))
                return
            for b in content or []:
                if not isinstance(b, dict):
                    continue
                if b.get("type") == "tool_result":
                    tid = b.get("tool_use_id")
                    if a.tool_names.get(tid) == "SubagentHandback":
                        continue
                    a.events.append((ts, "result", (tid, a.tool_names.get(tid, "?"),
                                                    result_text(b.get("content")), bool(b.get("is_error")))))
                elif b.get("type") == "text" and b.get("text"):
                    a.events.append((ts, "note", b["text"]))
            return
        m = e.get("message", {})
        u = m.get("usage")
        if u:
            a.context = context_of(u)
        for blk in m.get("content", []):
            kind = blk.get("type")
            if kind == "text" and blk.get("text", "").strip():
                if not a.handed_back:
                    a.report = blk["text"]
                a.events.append((ts, "text", blk["text"]))
            elif kind == "thinking" and blk.get("thinking", "").strip():
                a.events.append((ts, "thinking", blk["thinking"]))
            elif kind == "tool_use" and blk["name"] == "SubagentHandback":
                a.handed_back = True
                a.tool_names[blk.get("id")] = "SubagentHandback"
                a.report = str(blk.get("input", {}).get("message", ""))
                a.events.append((ts, "report", a.report))
            elif kind == "tool_use":
                name, inp, tid = blk["name"], blk.get("input", {}), blk.get("id")
                a.tools += 1
                a.tool_names[tid] = name
                a.last_tool = f'{name} {short_input(name, inp)}'.strip()
                a.events.append((ts, "tool", (name, short_input(name, inp), inp, tid)))
                if name == "Skill":
                    a.skills.append(inp.get("skill", "?"))
                self._tool(ts, f"agent:{a.id[:6]}", name, inp, tid)

    # ---- brief vs result ----------------------------------------------------
    def report_of(self, a):
        if a.report.strip():
            return a.report
        res = self.results.get(a.tool_use_id)  # a foreground agent's report is the Agent call's result
        if res and not res[1].startswith("Async agent launched"):
            return res[1]
        return ""

    def usage(self, a):
        """Heuristic: how much of what the agent found shows up in what the main thread wrote next.

        Terms that were already in the brief don't count: the parent knew them before.
        The window runs from the hand-back (or the agent's last event) to the next user prompt.
        """
        report = self.report_of(a)
        key = (a.id, a.finished, len(report), len(self.main_out), len(self.prompt_ts))
        cached = getattr(a, "_usage", None)
        if cached and cached[0] == key:
            return cached[1]
        res = {"verdict": "running", "score": None, "used": [], "unused": [], "window": 0}
        if a.finished:
            start = self.handback.get(a.id) or a.end
            stop = next((p for p in self.prompt_ts if p > start), None)
            later = [txt for ts, txt in self.main_out if ts >= start and (stop is None or ts < stop)]
            res["window"] = len(later)
            hay = "\n".join(later).lower()
            terms = report_terms(report, a.brief)
            for term, weight in terms:
                (res["used"] if term.lower() in hay else res["unused"]).append((term, weight))
            total = sum(w for _, w in terms)
            if not report.strip():
                res["verdict"] = "no report"
            elif not later:
                res["verdict"] = "not read yet"
            elif not total:
                res["verdict"] = "nothing new"
            else:
                res["score"] = sum(w for _, w in res["used"]) / total
                res["verdict"] = "used" if res["score"] >= 0.3 else "partly" if res["score"] >= 0.1 else "ignored?"
        a._usage = (key, res)
        return res


STOPWORDS = set("""
about above according actually additional already although another anything because before being below
between change changes changed check checked could currently different directly during either everything
example exactly existing following function functions however instead itself likely might nothing number
others otherwise output possible probably question really result results returns second should similar
something specific still summary through together though under until using whether which within without
would yourself recommend recommendation important including""".split())


def report_terms(report, brief, cap=60):
    """Distinctive terms of a report that were not in the brief, as (term, weight).

    Weight 2: paths, file names, `code spans`, identifiers. Weight 1: long plain words.
    """
    known = brief.lower()
    seen, out = set(), []

    def add(term, w):
        k = term.lower().strip(".,:;()[]'\"")
        if len(k) < 4 or k in seen or k in known:
            return
        seen.add(k)
        out.append((k, w))

    for m in re.findall(r"`([^`\n]{3,60})`", report):
        add(m, 2)
    for m in re.findall(r"[\w.~-]+/[\w./-]*\w|\b[\w-]+\.(?:py|js|ts|tsx|jsx|md|json|jsonl|ya?ml|toml|go|rs|sh|sql|css|html)\b", report):
        add(m, 2)
    for m in re.findall(r"\b[a-z]+_[a-z0-9_]+\b|\b[a-z]+[A-Z][A-Za-z0-9]+\b|\b[A-Z][a-z]+[A-Z][A-Za-z0-9]+\b", report):
        add(m, 2)
    for m in re.findall(r"\b[A-Za-z]{8,}\b", report):
        if m.lower() not in STOPWORDS:
            add(m, 1)
    out.sort(key=lambda x: -x[1])
    return out[:cap]


def brief_hints(brief):
    """A few cheap, honest signals about a brief."""
    words = len(brief.split())
    files = len(set(re.findall(r"[\w.~-]*/[\w./-]+\.\w+|\b[\w-]+\.(?:py|js|ts|tsx|md|json|ya?ml|toml|go|rs|sh)\b", brief)))
    asks = bool(re.search(r"\b(return|report|respond|reply|give me|list|summar|output|format|deliver)", brief, re.I))
    return words, files, asks


CHARS_PER_TOKEN = 3.7      # rough average for English, code and JSON
IMAGE_TOKENS = 1600        # a typical screenshot; base64 size says nothing useful

CATEGORIES = ["system prompt", "tool definitions", "deferred tools", "reminders & attachments", "user prompts",
              "assistant replies", "thinking", "tool calls", "tool results", "images", "not recorded"]


def block_text(b):
    """The part of a content block that takes up context, as text."""
    t = b.get("type")
    if t == "text":
        return b.get("text", "")
    if t == "thinking":
        return b.get("thinking", "")
    if t == "tool_use":
        return json.dumps(b.get("input", {}), ensure_ascii=False)
    if t == "tool_result":
        c = b.get("content")
        if isinstance(c, list):
            return "\n".join(x.get("text", "") for x in c if isinstance(x, dict) and x.get("type") == "text")
        return str(c or "")
    return json.dumps(b, ensure_ascii=False)


def context_items(shape, msgs):
    """Everything in one API request as dicts: cat, label, tokens (estimated), text, where."""
    items = []

    def add(cat, label, text, where, tokens=None):
        items.append({"cat": cat, "label": label, "text": text, "where": where,
                      "tokens": tokens if tokens is not None else len(text) / CHARS_PER_TOKEN})

    for i, b in enumerate(shape.get("system") or []):
        text = b.get("text", "") if isinstance(b, dict) else str(b)
        add("system prompt", f"system block {i + 1} · " + text.strip().split("\n")[0][:60], text, "system")
    for tdef in shape.get("tools") or []:
        add("tool definitions", tdef.get("name", "?"), json.dumps(tdef, indent=1, ensure_ascii=False), "tools")

    names = {}  # tool_use id -> (name, short)
    for mi, m in enumerate(msgs):
        where = f"msg {mi + 1}"
        if m is None:
            add("not recorded", "message not found in the transcript", "", where, 0)
            continue
        role, content = m.get("role"), m.get("content")
        blocks = content if isinstance(content, list) else [{"type": "text", "text": str(content or "")}]
        for b in blocks:
            if not isinstance(b, dict):
                continue
            t = b.get("type")
            if t == "tool_addition":
                d = b.get("tool", {})
                d = d.get("definition", d) if isinstance(d, dict) else {}
                add("deferred tools", d.get("name", "?"), json.dumps(d, indent=1, ensure_ascii=False), where)
            elif t in ("image", "document"):
                add("images", t, f"[{t}]", where, IMAGE_TOKENS)
            elif t == "tool_use":
                name, inp = b.get("name", "?"), b.get("input", {})
                names[b.get("id")] = (name, short_input(name, inp))
                add("tool calls", f"{name} · {short_input(name, inp)}", json.dumps(inp, indent=1, ensure_ascii=False), where)
            elif t == "tool_result":
                name, short = names.get(b.get("tool_use_id"), ("?", ""))
                add("tool results", f"{name} · {short}" + (" · ERROR" if b.get("is_error") else ""), block_text(b), where)
                for x in b.get("content") if isinstance(b.get("content"), list) else []:
                    if isinstance(x, dict) and x.get("type") in ("image", "document"):
                        add("images", f"{x['type']} in {name} result", f"[{x['type']}]", where, IMAGE_TOKENS)
            elif t == "thinking" or t == "redacted_thinking":
                text = block_text(b)
                add("thinking", text.strip().split("\n")[0][:70] or "(hidden: only the signature is stored)", text, where)
            elif t == "text":
                text = b.get("text", "")
                first = text.strip().split("\n")[0][:70]
                if role == "assistant":
                    add("assistant replies", first, text, where)
                elif role == "system" or text.lstrip().startswith("<") or not is_real_prompt(text):
                    tag = re.match(r"\s*<([\w-]+)", text)
                    add("reminders & attachments", (f"<{tag.group(1)}> " if tag else "") + first[:60], text, where)
                else:
                    add("user prompts", first, text, where)
            else:
                add("reminders & attachments", t or "?", block_text(b), where)
    return items


def elapsed(start, end):
    try:
        a = datetime.fromisoformat(start.replace("Z", "+00:00"))
        b = datetime.fromisoformat(end.replace("Z", "+00:00"))
        sec = int((b - a).total_seconds())
        return f"{sec // 60}m{sec % 60:02d}s" if sec >= 60 else f"{sec}s"
    except Exception:
        return "—"


def bar(frac, width):
    frac = max(0.0, min(1.0, frac))
    n = int(round(frac * width))
    color = GREEN if frac < 0.5 else YELLOW if frac < 0.8 else RED
    return c(color, "█" * n) + c(DIM, "░" * (width - n))


def render(s, limit):
    if not limit:  # auto: a session past 200k must be running with the 1M window
        limit = context_window(s.model, s.peak)
    cols = shutil.get_terminal_size((100, 40)).columns
    w = max(60, min(cols, 120))
    L = []
    hr = c(DIM, "─" * w)
    L.append(c(BOLD, f"CLAUDE X-RAY") + c(DIM, f"  session {s.id[:8]}  ·  {s.path.parent.name[:50]}  ·  {datetime.now():%H:%M:%S}"))
    if s.last_prompt:
        L.append(c(DIM, "last prompt: ") + s.last_prompt[: w - 16])
    L.append(hr)

    # context
    frac = s.context / limit if limit else 0
    L.append(c(BOLD, "CONTEXT ") + f"{fmt_tok(s.context)} / {fmt_tok(limit)}  ({frac:.0%})  "
             + c(DIM, f"peak {fmt_tok(s.peak)} · total output {fmt_tok(s.out_tokens)} · compactions {s.compactions} · model {s.model or '?'}"))
    L.append("  " + bar(frac, w - 4))
    L.append(hr)

    # agents
    running = [a for a in s.agents.values() if not a.finished]
    done = [a for a in s.agents.values() if a.finished]
    L.append(c(BOLD, "AGENTS ") + f"launched {s.agent_calls}  ·  "
             + c(YELLOW, f"running {len(running)}") + "  ·  " + c(GREEN, f"finished {len(done)}"))
    show = sorted(running, key=lambda a: a.start, reverse=True) + sorted(done, key=lambda a: a.start, reverse=True)
    for a in show[:8]:
        st = c(YELLOW, "● running ") if not a.finished else c(GREEN, "✓ done     ")
        sk = c(MAGENTA, " skills:" + ",".join(dict.fromkeys(a.skills))) if a.skills else ""
        u = s.usage(a)
        used = f"  {u['verdict']}" + (f" {u['score']:.0%}" if u["score"] is not None else "") if a.finished else ""
        line = f"  {st} {a.desc[:34]:34} {c(DIM, (a.type or '')[:16]):16} ctx {fmt_tok(a.context):>6}  tools {a.tools:>3}{used}  {c(DIM, a.last_tool[:30])}{sk}"
        L.append(line)
    if len(show) > 8:
        L.append(c(DIM, f"  … and {len(show) - 8} more"))
    L.append(hr)

    # skills
    L.append(c(BOLD, "SKILLS ") + f"available {s.skills_available}  ·  invoked {len(s.skill_calls)}"
             + (f"  ·  new this session {len(s.skills_added)}" if s.skills_added else ""))
    for ts, who, name, how, *_ in s.skill_calls[-6:]:
        L.append(f"  {c(DIM, hhmm(ts))}  {c(MAGENTA, name):40} {c(DIM, how)}  {c(DIM, who)}")
    for ts, name in s.skills_added[-3:]:
        L.append(f"  {c(DIM, hhmm(ts))}  {c(CYAN, '+ ' + name)}  {c(DIM, 'added to listing')}")
    if s.skill_reads:
        reads = "  ·  ".join(f"{k}: {', '.join(sorted(v))}" for k, v in list(s.skill_reads.items())[-3:])
        L.append(c(DIM, "  skill files read → ") + reads[: w - 28])
    L.append(hr)

    # feed
    L.append(c(BOLD, "RECENT ACTIVITY"))
    for ts, who, name, detail, *_ in sorted(s.feed, key=lambda x: x[0])[-10:]:
        whoc = c(CYAN, f"{who:12}") if who != "main" else c(DIM, f"{who:12}")
        L.append(f"  {c(DIM, hhmm(ts))}  {whoc} {c(BOLD, f'{name[:10]:10}')} {detail[: w - 38]}")
    return "\n".join(L)


def last_prompt_of(f, tail_bytes=400_000):
    """Last real user prompt, scanning only the end of the file."""
    try:
        with open(f, "rb") as fh:
            size = fh.seek(0, 2)
            fh.seek(max(0, size - tail_bytes))
            lines = fh.read().decode("utf-8", "replace").split("\n")
    except OSError:
        return ""
    for l in reversed(lines):
        if '"type":"user"' not in l:
            continue
        try:
            e = json.loads(l)
        except Exception:
            continue
        content = e.get("message", {}).get("content")
        content = first_text(content)
        if content.strip() and is_real_prompt(content):
            return content.strip().replace("\n", " ")
    return ""


def list_sessions(all_projects, cwd, n, headless=False):
    return sorted(session_files(all_projects, cwd, headless), key=lambda f: f.stat().st_mtime, reverse=True)[:n]


def resolve_session(arg):
    p = Path(arg)
    if p.exists():
        return p
    hits = list(PROJECTS.glob(f"*/{arg}*.jsonl"))
    return hits[0] if hits else None


# ---------------------------------------------------------------- Textual UI
from rich.console import Group
from rich.markdown import Markdown
from rich.rule import Rule
from rich.syntax import Syntax
from rich.text import Text
from textual.app import App, ComposeResult
from textual.containers import Horizontal, Vertical, VerticalScroll
from textual.screen import ModalScreen
from textual.widgets import DataTable, Footer, Header, Label, RichLog, Sparkline, Static

AUTO = "__auto__"


def kv(*pairs):
    """Aligned 'label  value' lines."""
    t = Text()
    for k, v in pairs:
        t.append(f"{k:>14}  ", style="dim")
        t.append(v if isinstance(v, Text) else Text(str(v)))
        t.append("\n")
    return t


def code(text, lexer):
    return Syntax(text, lexer, theme="monokai", word_wrap=True, background_color="default")


def result_block(s, tid):
    if not tid or tid not in s.results:
        return Text("(no result yet)", style="dim italic")
    ts, text, err = s.results[tid]
    head = Text(f"{hhmm(ts)}  ", style="dim") + (Text("ERROR", style="bold red") if err else Text("ok", style="green"))
    return Group(head, Text(text or "(empty)", style="red" if err else ""))


class Detail(ModalScreen):
    """Scrollable read-only dialog."""
    BINDINGS = [("escape", "close", "close"), ("q", "close", "close")]

    def __init__(self, title, body):
        super().__init__()
        self.box_title, self.body = title, body

    def compose(self) -> ComposeResult:
        with VerticalScroll(id="detail", classes="dialog") as v:
            v.border_title = self.box_title
            v.border_subtitle = "Esc closes · ↑↓ PgUp PgDn scroll"
            yield Static(self.body)

    def on_mount(self):
        self.query_one("#detail").focus()

    def action_close(self):
        self.dismiss()


class ContextDetail(ModalScreen):
    BINDINGS = [("escape", "close", "close"), ("q", "close", "close"), ("d", "inside", "what's in context")]

    def __init__(self, s, limit):
        super().__init__()
        self.s, self.limit = s, limit

    def compose(self) -> ComposeResult:
        s, h = self.s, self.s.ctx_hist
        with Vertical(id="ctxdialog", classes="dialog") as v:
            v.border_title = f"Context · session {s.id[:8]}"
            v.border_subtitle = "d what's in the context now · Enter what was in it for that response · Esc closes"
            last = h[-1] if h else None
            frac = s.context / self.limit if self.limit else 0
            yield Static(kv(
                ("current", Text(f"{s.context:,} tokens  ({frac:.1%} of {fmt_tok(self.limit)})", style="bold")),
                ("remaining", f"{max(0, self.limit - s.context):,} tokens"),
                ("peak", f"{s.peak:,}"),
                ("last call", f"input {last[2]:,} · cache read {last[3]:,} · cache created {last[4]:,} · output {last[5]:,}"
                 if last else "—"),
                ("total output", f"{s.out_tokens:,} tokens in {len(h)} responses"),
                ("compactions", ", ".join(hhmm(t) for t in s.compact_ts) or "none"),
                ("model", s.model or "?"),
            ))
            yield Label(Text("context over time (one bar per model response)", style="dim"))
            yield Sparkline([r[1] for r in h] or [0], id="spark")
            yield DataTable(id="ctxtable", cursor_type="row", zebra_stripes=True)

    def on_mount(self):
        t = self.query_one("#ctxtable", DataTable)
        t.add_columns("time", "context", "Δ", "input", "cache read", "cache created", "output", "turn prompt")
        prev = 0
        rows = []
        for i, (ts, ctx, inp, cr, cn, out, prompt, mid) in enumerate(self.s.ctx_hist):
            d = ctx - prev
            prev = ctx
            dt = Text(f"{'+' if d >= 0 else ''}{fmt_tok(abs(d)) if d >= 0 else '-' + fmt_tok(-d)}",
                      style="red" if d > 20_000 else "yellow" if d > 5_000 else "green" if d < 0 else "dim")
            rows.append((str(i), (hhmm(ts), fmt_tok(ctx), dt, fmt_tok(inp), fmt_tok(cr), fmt_tok(cn), fmt_tok(out),
                                  Text(prompt[:70], style="dim"))))
        for key, r in reversed(rows):
            t.add_row(*r, key=key)
        t.focus()

    def open_inside(self, req, label):
        if req is None:
            self.app.notify("No request log for this session: it was recorded by an older Claude Code "
                            "version, or this response has no logged request.", severity="warning", timeout=6)
            return
        self.app.push_screen(ContextContents(self.s, req, label))

    def action_inside(self):
        self.open_inside(self.s.request_for(), "now (latest request)")

    def on_data_table_row_selected(self, ev: DataTable.RowSelected):
        ev.stop()
        ts, ctx, *_, mid = self.s.ctx_hist[int(ev.row_key.value)]
        self.open_inside(self.s.request_for(mid), f"response at {hhmm(ts)} · {fmt_tok(ctx)}")

    def action_close(self):
        self.dismiss()


CAT_STYLE = {"system prompt": "blue", "tool definitions": "magenta", "deferred tools": "medium_purple",
             "reminders & attachments": "cyan", "user prompts": "bold white", "assistant replies": "green",
             "thinking": "grey62", "tool calls": "yellow", "tool results": "dark_orange", "images": "pink1",
             "not recorded": "red"}


class ContextContents(ModalScreen):
    """What one API request carried: system prompt, tools, every message block, with sizes."""
    BINDINGS = [("escape", "close", "close"), ("q", "close", "close"), ("o", "sort", "order/size"),
                ("g", "group", "group by category")]

    def __init__(self, s, req, label):
        super().__init__()
        self.s, self.req, self.label = s, req, label
        self.by_size, self.grouped = True, False
        shape, msgs = s.request_context(req)
        self.items = context_items(shape, msgs)
        self.n_msgs = len(msgs)
        est = sum(i["tokens"] for i in self.items)
        self.real = s.real_context(req)
        # scale so the parts add up to what the API reported; the split stays an estimate
        self.scale = self.real / est if self.real and est else 1.0
        for i in self.items:
            i["tokens"] *= self.scale

    def compose(self) -> ComposeResult:
        with Vertical(id="ccdialog", classes="dialog"):
            yield Static(id="ccsum")
            yield DataTable(id="cctable", cursor_type="row", zebra_stripes=True)

    def on_mount(self):
        box = self.query_one("#ccdialog")
        box.border_title = f"Inside the context · {self.label}"
        box.border_subtitle = "Enter opens an item · o order/size · g group by category · Esc closes"
        self.query_one("#ccsum", Static).update(self.summary())
        t = self.query_one("#cctable", DataTable)
        t.add_columns("≈ tokens", "%", "category", "item", "where")
        self.fill()
        t.focus()

    def summary(self):
        total = sum(i["tokens"] for i in self.items) or 1
        cats = {}
        for i in self.items:
            c = cats.setdefault(i["cat"], [0, 0])
            c[0] += i["tokens"]
            c[1] += 1
        p = self.req.get("params") or {}
        out = Text()
        out.append(f"{fmt_tok(int(total))} tokens", style="bold")
        out.append("  as reported by the API" if self.real else "  estimated from characters (no response yet)", style="dim")
        out.append(f" · {len(self.items)} items in {self.n_msgs} messages · request {hhmm(self.req.get('timestamp', ''))}"
                   f" · {self.req.get('querySource', '?')} · {p.get('model', '?')}\n", style="dim")
        width = 46
        for name in CATEGORIES:
            if name not in cats:
                continue
            tok, n = cats[name]
            frac = tok / total
            out.append(f"{name:>24} ", style=CAT_STYLE[name])
            out.append("█" * max(1, int(round(frac * width))) if tok else "", style=CAT_STYLE[name])
            out.append(f" {fmt_tok(int(tok))} · {frac:.0%} · {n} item{'s' * (n != 1)}", style="dim")
            if name == "tool results":
                per = {}
                for i in self.items:
                    if i["cat"] == name:
                        per[i["label"].split(" · ")[0]] = per.get(i["label"].split(" · ")[0], 0) + i["tokens"]
                top = sorted(per.items(), key=lambda kv: -kv[1])[:4]
                out.append("  (" + ", ".join(f"{k} {fmt_tok(int(v))}" for k, v in top) + ")", style="dim")
            out.append("\n")
        cm = (p.get("context_management") or {}).get("edits")
        if cm:
            out.append("context editing: " + ", ".join(e.get("type", "?") for e in cm) + "\n", style="dim")
        out.append("The split between items is estimated from character counts"
                   + (", scaled to the real total." if self.real else ".") +
                   " The content is exactly what Claude Code sent.", style="dim italic")
        return out

    def fill(self):
        t = self.query_one("#cctable", DataTable)
        t.clear()
        total = sum(i["tokens"] for i in self.items) or 1
        rows = list(enumerate(self.items))
        if self.grouped:
            rows.sort(key=lambda r: (CATEGORIES.index(r[1]["cat"]), -r[1]["tokens"]))
        elif self.by_size:
            rows.sort(key=lambda r: -r[1]["tokens"])
        for idx, i in rows:
            tok = i["tokens"]
            style = "bold red" if tok > 10_000 else "yellow" if tok > 2_000 else ""
            t.add_row(Text(fmt_tok(int(tok)), style=style), Text(f"{tok / total:.1%}", style="dim"),
                      Text(i["cat"], style=CAT_STYLE[i["cat"]]), Text(i["label"][:70]),
                      Text(i["where"], style="dim"), key=str(idx))

    def on_data_table_row_selected(self, ev: DataTable.RowSelected):
        ev.stop()
        i = self.items[int(ev.row_key.value)]
        text = i["text"] or "(empty)"
        body = cap(text, 60_000, hint="")
        lexer = "json" if i["cat"] in ("tool definitions", "deferred tools", "tool calls") else None
        head = kv(("category", Text(i["cat"], style=CAT_STYLE[i["cat"]])), ("item", i["label"]),
                  ("where", i["where"]), ("size", f"≈ {fmt_tok(int(i['tokens']))} tokens · {len(text):,} chars"))
        self.app.push_screen(Detail(f"{i['cat']} · {i['label'][:60]}",
                                    Group(head, Rule(style="dim"), code(body, lexer) if lexer else Text(body))))

    def action_sort(self):
        self.grouped, self.by_size = False, not self.by_size
        self.fill()

    def action_group(self):
        self.grouped = not self.grouped
        self.fill()

    def action_close(self):
        self.dismiss()


def cap(text, n, hint=", v shows all"):
    return text if len(text) <= n else text[:n] + f"\n… (+{len(text) - n} chars{hint})"


def now_iso():
    return datetime.now().astimezone().isoformat()


def agent_order(s):
    return sorted(s.agents.values(), key=lambda a: a.start or "")


VERDICT_STYLE = {"used": ("✓", "bold green"), "partly": ("~", "bold yellow"), "ignored?": ("✗", "bold red"),
                 "not read yet": ("…", "yellow"), "nothing new": ("=", "dim"), "no report": ("∅", "dim"),
                 "running": ("●", "yellow")}


def verdict_text(u, long=False, short=False):
    icon, style = VERDICT_STYLE.get(u["verdict"], ("?", "dim"))
    pct = f" {u['score']:.0%}" if u["score"] is not None else ""
    if short:
        return Text(f"{icon}{pct}" if pct else f"{icon} {u['verdict']}", style=style)
    label = u["verdict"].upper() if long else u["verdict"]
    return Text(f"{icon} {label}{pct}", style=style)


class AgentFocus(ModalScreen):
    """tail -f of one subagent: brief, thinking, text, tool calls, results and report as they arrive."""
    BINDINGS = [("escape", "close", "close"), ("q", "close", "close"), ("f", "follow", "follow on/off"),
                ("v", "verbose", "full output"), ("b", "brief", "brief vs result"),
                ("left_square_bracket", "cycle(-1)", "prev agent"), ("right_square_bracket", "cycle(1)", "next agent")]

    def __init__(self, s, a):
        super().__init__()
        self.s, self.a = s, a
        self.n, self.verbose, self.marked = 0, False, False

    def compose(self) -> ComposeResult:
        with Vertical(id="focus", classes="dialog"):
            yield Static(id="fstatus")
            yield RichLog(id="flog", wrap=True, auto_scroll=True, min_width=40)

    def on_mount(self):
        self.reset()
        self.set_interval(0.5, self.pull)
        self.query_one("#flog").focus()

    def reset(self):
        self.query_one("#flog", RichLog).clear()
        self.n, self.marked = 0, False
        box = self.query_one("#focus")
        box.border_title = f"Focus · {self.a.desc[:70] or self.a.id}"
        box.border_subtitle = "f follow · v full output · b brief vs result · [ ] prev/next agent · Esc closes"
        self.pull()

    def pull(self):
        a, log = self.a, self.query_one("#flog", RichLog)
        for ev in a.events[self.n:]:
            log.write(self.render_event(*ev))
        self.n = len(a.events)
        if a.finished and not self.marked:
            log.write(Rule(f"finished {hhmm(a.end)}", style="green"))
            self.marked = True
        status = Text("✓ done", style="bold green") if a.finished else Text("● running", style="bold yellow")
        self.query_one("#fstatus", Static).update(Text.assemble(
            status, ("  ·  ", "dim"), (a.type or "agent", "cyan"), ("  ·  ", "dim"), a.model or "?",
            ("  ·  ctx ", "dim"), fmt_tok(a.context), ("  ·  tools ", "dim"), str(a.tools),
            ("  ·  ", "dim"), elapsed(a.start, a.end if a.finished else now_iso()),
            ("  ·  follow ", "dim"), ("on", "green") if log.auto_scroll else ("off", "yellow"),
            ("  ·  ", "dim"), ("full output" if self.verbose else "compact", "dim italic")))

    def render_event(self, ts, kind, data):
        when = Text(f"{hhmm(ts)}  ", style="dim")
        if kind == "brief":
            return Group(Rule("brief from the parent", style="cyan"),
                         Text(data if self.verbose else cap(data, 1500)), Rule(style="cyan"))
        if kind == "report":
            return Group(Rule("report handed back to the parent", style="green"), Markdown(data), Rule(style="green"))
        if kind == "text":
            return Group(when + Text("says", style="bold"), Markdown(data if self.verbose else cap(data, 3000)))
        if kind == "thinking":
            return when + Text("thinking  " + (data if self.verbose else cap(data, 400)), style="dim italic")
        if kind == "tool":
            name, short, inp, _ = data
            line = when + Text("→ ", style="cyan") + Text(name, style="bold cyan") + Text(f"  {short}")
            if self.verbose:
                return Group(line, code(json.dumps(inp, indent=2, ensure_ascii=False)[:5000], "json"))
            if name == "Bash" and inp.get("command") and inp.get("command") != short:
                line.append(f"\n           $ {inp['command'][:300]}", style="dim")
            return line
        if kind == "result":
            _, name, text, err = data
            lines = text.splitlines()
            head = Text("           ← ", style="dim") + (Text("ERROR", style="bold red") if err else Text("ok", style="green"))
            head.append(f"  {name} · {len(lines)} lines", style="dim")
            body = text if self.verbose else "\n".join(lines[:6]) + (f"\n… (+{len(lines) - 6} lines)" if len(lines) > 6 else "")
            return Group(head, Text(body, style="red" if err else "grey50"))
        # note: anything else the agent received mid-run
        if data.startswith("<system-reminder"):
            return when + Text(f"· system reminder ({len(data)} chars)", style="dim")
        return when + Text("· message in: ", style="magenta") + Text(data if self.verbose else cap(data, 600))

    def action_follow(self):
        log = self.query_one("#flog", RichLog)
        log.auto_scroll = not log.auto_scroll
        if log.auto_scroll:
            log.scroll_end(animate=False)
        self.pull()

    def action_verbose(self):
        self.verbose = not self.verbose
        self.reset()

    def action_cycle(self, step):
        order = agent_order(self.s)
        if len(order) > 1 and self.a in order:
            self.a = order[(order.index(self.a) + step) % len(order)]
            self.reset()

    def action_brief(self):
        self.app.push_screen(BriefResult(self.s, self.a))

    def action_close(self):
        self.dismiss()


class BriefResult(ModalScreen):
    """What the parent asked, what the agent returned, and whether the parent used it."""
    BINDINGS = [("escape", "close", "close"), ("q", "close", "close"),
                ("left_square_bracket", "cycle(-1)", "prev agent"), ("right_square_bracket", "cycle(1)", "next agent")]

    def __init__(self, s, a):
        super().__init__()
        self.s, self.a, self._sig = s, a, None

    def compose(self) -> ComposeResult:
        with Vertical(id="brdialog", classes="dialog"):
            yield Static(id="brhead")
            with Horizontal(id="brcols"):
                with VerticalScroll(classes="box", id="brbrief"):
                    yield Static(id="brbrieftext")
                with VerticalScroll(classes="box", id="brreport"):
                    yield Static(id="brreporttext")

    def on_mount(self):
        self.query_one("#brdialog").border_subtitle = "Tab switches pane · [ ] prev/next agent · Esc closes"
        self.query_one("#brbrief").border_title = "Brief (what the parent asked)"
        self.query_one("#brreport").border_title = "Result (what came back)"
        self.redraw()
        self.set_interval(1.0, self.redraw)
        self.query_one("#brreport").focus()

    def redraw(self):
        s, a = self.s, self.a
        report = s.report_of(a)
        u = s.usage(a)
        sig = (a.id, a.finished, len(report), u["verdict"], u["score"], a.tools, a.context)
        if sig == self._sig:
            return
        self._sig = sig
        self.query_one("#brdialog").border_title = f"Brief vs result · {a.desc[:70] or a.id}"
        words, files, asks = brief_hints(a.brief)
        head = Text()
        head.append_text(verdict_text(u, long=True))
        explain = {
            "running": "  the agent is still working",
            "no report": "  the agent finished without a report",
            "not read yet": "  the main thread has written nothing since the hand-back",
            "nothing new": "  the report holds no terms that weren't already in the brief",
        }.get(u["verdict"], f"  share of the report's new terms that show up in the main thread's next "
                            f"{u['window']} messages (until the next user prompt). A heuristic, not proof.")
        head.append(explain + "\n", style="dim")
        if u["used"]:
            head.append("used     ", style="dim")
            head.append("  ".join(t for t, _ in u["used"][:25]) + "\n", style="green")
        if u["unused"]:
            head.append("unused   ", style="dim")
            head.append("  ".join(t for t, _ in u["unused"][:25]) + "\n", style="grey50")
        head.append("brief    ", style="dim")
        head.append(f"{words} words · names {files} file{'s' * (files != 1)} · ")
        head.append("asks for a specific output" if asks else "doesn't say what to return",
                    style="" if asks else "yellow")
        head.append("\nresult   ", style="dim")
        head.append(f"{len(report.split())} words · ctx {fmt_tok(a.context)} · {a.tools} tools · "
                    f"{elapsed(a.start, a.end if a.finished else now_iso())}")
        if a.handed_back or s.handback.get(a.id):
            head.append(f" · handed back {hhmm(s.handback.get(a.id) or a.end)}", style="dim")
        self.query_one("#brhead", Static).update(head)
        self.query_one("#brbrieftext", Static).update(Text(a.brief or "(brief not recorded)"))
        self.query_one("#brreporttext", Static).update(
            Markdown(report) if report.strip() else Text("(no report yet)", style="dim italic"))

    def action_cycle(self, step):
        order = agent_order(self.s)
        if len(order) > 1 and self.a in order:
            self.a = order[(order.index(self.a) + step) % len(order)]
            self._sig = None
            self.redraw()

    def action_close(self):
        self.dismiss()


class Picker(ModalScreen):
    """Lists recent sessions. Dismisses with a Path, AUTO (follow newest) or None (cancel)."""
    BINDINGS = [("escape", "cancel", "back"), ("a", "toggle_all", "all projects"),
                ("h", "toggle_headless", "headless runs")]

    def __init__(self, app_args, can_cancel):
        super().__init__()
        self.args, self.can_cancel = app_args, can_cancel

    def compose(self) -> ComposeResult:
        with Vertical(id="picker", classes="dialog"):
            yield DataTable(id="ptable", cursor_type="row", zebra_stripes=True)
            yield Label(Text("Enter opens · a toggles current/all projects · h shows/hides headless `claude -p` runs · Esc goes back", style="dim"))

    def on_mount(self):
        self.load()

    def load(self):
        box = self.query_one("#picker")
        box.border_title = ("Recent sessions · " + ("all projects" if self.args.all else "current project")
                            + (" · including headless runs" if self.args.headless else ""))
        t = self.query_one(DataTable)
        t.clear(columns=True)
        cols = ["when", "session"] + (["project"] if self.args.all else []) + ["last prompt"]
        t.add_columns(*cols)
        auto = [Text("⟳ follow the newest", style="bold cyan"), Text("auto", style="dim")]
        auto += [""] * (len(cols) - len(auto) - 1) + [Text("switches automatically after /clear", style="dim")]
        t.add_row(*auto, key=AUTO)
        files = list_sessions(self.args.all, self.args.cwd, self.args.list, self.args.headless)
        for f in files:
            sid = Text(f.stem[:8]) + (Text(" -p", style="dim") if is_headless(f) else Text(""))
            row = [datetime.fromtimestamp(f.stat().st_mtime).strftime("%m-%d %H:%M"), sid]
            if self.args.all:
                row.append(f.parent.name[-30:])
            row.append(Text(last_prompt_of(f)[:90] or "(no prompts)"))
            t.add_row(*row, key=str(f))
        if not files and not self.args.headless:
            hidden = len(session_files(self.args.all, self.args.cwd, headless=True))
            if hidden:
                hint = [""] * (len(cols) - 1) + [Text(f"{hidden} headless `claude -p` runs hidden · h shows them",
                                                       style="dim italic")]
                t.add_row(*hint, key="__hint__")
        t.focus()

    def on_data_table_row_selected(self, ev: DataTable.RowSelected):
        ev.stop()
        k = ev.row_key.value
        if k == "__hint__":
            return self.action_toggle_headless()
        self.dismiss(AUTO if k == AUTO else Path(k))

    def action_toggle_headless(self):
        self.args.headless = not self.args.headless
        self.load()

    def action_toggle_all(self):
        self.args.all = not self.args.all
        self.load()

    def action_cancel(self):
        if self.can_cancel:
            self.dismiss(None)
        else:
            self.app.exit()


class XRay(App):
    TITLE = "Claude X-Ray"
    CSS = """
    #prompt, #ctx, #ctxbar { height: 1; padding: 0 1; }
    #ctx:hover, #ctxbar:hover { background: $boost; }
    #ctxbar { margin-bottom: 1; }
    #mid { height: 13; }
    .box { border: round $primary-darken-2; border-title-color: $text; border-title-style: bold; }
    .box:focus-within { border: round $accent; }
    #agentsbox { width: 3fr; }
    #skillsbox { width: 2fr; }
    #feedbox { height: 1fr; }
    DataTable { height: 1fr; }
    Picker, Detail, ContextDetail, AgentFocus, BriefResult { align: center middle; }
    #fstatus { height: 1; margin-bottom: 1; }
    #flog { height: 1fr; background: $surface; }
    #brhead { height: auto; max-height: 14; margin-bottom: 1; }
    #brcols { height: 1fr; }
    #brbrief, #brreport { width: 1fr; padding: 0 1; }
    .dialog { width: 92%; height: 88%; border: thick $accent; background: $surface; padding: 0 1;
              border-title-style: bold; border-subtitle-color: $text-muted; }
    #spark { height: 4; margin-bottom: 1; }
    """
    BINDINGS = [("q", "quit", "quit"), ("s", "pick", "sessions"), ("c", "context", "context"),
                ("1", "focus('agents')", "agents"), ("2", "focus('skills')", "skills"),
                ("3", "focus('feed')", "activity"), ("f", "focus_agent", "focus agent"),
                ("b", "brief", "brief vs result"), ("p", "pause", "pause")]

    def __init__(self, args):
        super().__init__()
        self.args = args
        self.session, self.follow, self.paused, self._sig = None, False, False, None
        self.rows = {}   # row key -> payload for the detail dialogs

    def compose(self) -> ComposeResult:
        yield Header(show_clock=True)
        yield Static(id="prompt")
        yield Static(id="ctx")
        yield Static(id="ctxbar")
        with Horizontal(id="mid"):
            with Vertical(classes="box", id="agentsbox"):
                yield DataTable(id="agents", cursor_type="row", zebra_stripes=True)
            with Vertical(classes="box", id="skillsbox"):
                yield DataTable(id="skills", cursor_type="row", zebra_stripes=True)
        with Vertical(classes="box", id="feedbox"):
            yield DataTable(id="feed", cursor_type="row", zebra_stripes=True)
        yield Footer()

    def on_mount(self):
        self.query_one("#agents", DataTable).add_columns("", "description", "type", "ctx", "tools", "used?", "last tool", "skills")
        self.query_one("#skills", DataTable).add_columns("time", "skill", "how", "who")
        self.query_one("#feed", DataTable).add_columns("time", "who", "tool", "detail", "")
        self.query_one("#feedbox").border_title = "Recent activity · Enter opens details"
        self.query_one("#feed").focus()
        if self.args.session:
            path = resolve_session(self.args.session)
            if not path:
                self.exit(message=f"Session {self.args.session} not found.")
                return
            self.open(path)
        elif self.args.latest:
            self.follow = True
            self.open_newest()
        else:
            self.push_screen(Picker(self.args, can_cancel=False), self.picked)
        self.set_interval(self.args.interval, self.tick)

    # ---- session handling ------------------------------------------------
    def picked(self, result):
        if result is None:
            return
        if result == AUTO:
            self.follow = True
            self.open_newest()
        else:
            self.follow = False
            self.open(result)

    def open_newest(self):
        path = newest_session(self.args.all, self.args.cwd, self.args.headless)
        if path:
            self.open(path)
        else:
            self.session = None
            self.sub_title = "waiting for a session…"

    def open(self, path):
        self.session = Session(path)
        self.session.update()
        self._sig = None
        self.refresh_view()

    def tick(self):
        if self.paused:
            return
        if self.follow:
            newest = newest_session(self.args.all, self.args.cwd, self.args.headless)
            cur = self.session.path if self.session else None
            if newest and newest != cur and (cur is None or born(newest) > born(cur)
                                             or newest.stat().st_mtime > cur.stat().st_mtime + 2):
                self.open(newest)
                self.notify(f"Following new session {newest.stem[:8]}")
                return
        if self.session:
            self.session.update()
            self.refresh_view()

    # ---- actions -----------------------------------------------------------
    def action_pick(self):
        self.push_screen(Picker(self.args, can_cancel=True), self.picked)

    def action_pause(self):
        self.paused = not self.paused
        self.notify("Paused" if self.paused else "Resumed")

    def action_focus(self, wid):
        self.query_one(f"#{wid}").focus()

    def limit(self):
        return self.args.limit or context_window(self.session.model, self.session.peak)

    def action_context(self):
        if self.session:
            self.push_screen(ContextDetail(self.session, self.limit()))

    def selected_agent(self, prefer_running):
        """The agent under the cursor in the Agents table; otherwise the newest (running) one."""
        s = self.session
        if not s or not s.agents:
            return None
        t = self.query_one("#agents", DataTable)
        if self.focused is t and t.row_count:
            payload = self.rows.get(t.coordinate_to_cell_key(t.cursor_coordinate).row_key.value)
            if payload and payload[0] == "agent":
                return payload[1]
        order = agent_order(s)
        if prefer_running:
            running = [a for a in order if not a.finished]
            return (running or order)[-1]
        done = [a for a in order if a.finished]
        return (done or order)[-1]

    def action_focus_agent(self):
        a = self.selected_agent(prefer_running=True)
        if a:
            self.push_screen(AgentFocus(self.session, a))
        else:
            self.notify("No subagents in this session yet")

    def action_brief(self):
        a = self.selected_agent(prefer_running=False)
        if a:
            self.push_screen(BriefResult(self.session, a))
        else:
            self.notify("No subagents in this session yet")

    def on_click(self, ev):
        if ev.widget is not None and ev.widget.id in ("ctx", "ctxbar"):
            self.action_context()

    def on_data_table_row_selected(self, ev: DataTable.RowSelected):
        payload = self.rows.get(ev.row_key.value)
        if not payload or not self.session:
            return
        kind, data = payload
        if kind == "agent":
            self.push_screen(AgentFocus(self.session, data))
            return
        title, body = getattr(self, f"detail_{kind}")(data)
        self.push_screen(Detail(title, body))

    # ---- detail builders -----------------------------------------------------
    def detail_feed(self, entry):
        s = self.session
        ts, who, name, short, tid, inp = entry
        parts = [kv(("when", hhmm(ts)), ("who", who), ("tool", Text(name, style="bold")),
                    ("summary", short), ("id", tid or "—")),
                 Rule("input", style="dim")]
        if name == "Bash" and inp.get("command"):
            parts.append(code(inp["command"], "bash"))
            rest = {k: v for k, v in inp.items() if k != "command"}
            if rest:
                parts.append(code(json.dumps(rest, indent=2, ensure_ascii=False), "json"))
        else:
            parts.append(code(json.dumps(inp, indent=2, ensure_ascii=False), "json"))
        parts += [Rule("result", style="dim"), result_block(s, tid)]
        return f"{name} · {short[:60]}", Group(*parts)

    def detail_skillcall(self, call):
        s = self.session
        ts, who, name, how, tid, args = call
        parts = [kv(("skill", Text(name, style="bold magenta")), ("when", hhmm(ts)), ("how", how),
                    ("who", who), ("arguments", args or "—"))]
        parts += self._skill_info(name)
        if tid:
            parts += [Rule("call result", style="dim"), result_block(s, tid)]
        return f"Skill · {name}", Group(*parts)

    def detail_skillname(self, data):
        name, note = data
        return f"Skill · {name}", Group(kv(("skill", Text(name, style="bold magenta")), ("event", note)),
                                        *self._skill_info(name))

    def _skill_info(self, name):
        s = self.session
        base = name.split(":")[-1]
        reads = s.skill_reads.get(base) or s.skill_reads.get(name)
        out = []
        if reads:
            out.append(kv(("files read", ", ".join(sorted(reads)))))
        md = find_skill_md(name, s.cwd or self.args.cwd)
        if md:
            text = md.read_text(errors="replace")
            lines = text.splitlines()
            out += [Rule(str(md).replace(str(Path.home()), "~"), style="dim"),
                    code("\n".join(lines[:150]) + (f"\n\n… (+{len(lines) - 150} lines)" if len(lines) > 150 else ""),
                         "markdown")]
        else:
            out.append(Text("SKILL.md not found locally (built-in skill or remote plugin).", style="dim italic"))
        return out

    # ---- drawing -----------------------------------------------------------
    def fill(self, tid, rows):
        """Rebuild a table, keeping the cursor on the same item (or on top if it was there)."""
        t = self.query_one(f"#{tid}", DataTable)
        keep = None
        if t.row_count and t.cursor_row > 0:
            keep = t.coordinate_to_cell_key(t.cursor_coordinate).row_key.value
        t.clear()
        for key, cells, payload in rows:
            self.rows[key] = payload
            t.add_row(*cells, key=key)
        if keep is not None:
            try:
                t.move_cursor(row=t.get_row_index(keep), animate=False)
            except Exception:
                pass

    def refresh_view(self):
        s = self.session
        limit = self.limit()
        mode = "following newest" if self.follow else "pinned session"
        self.sub_title = f"{s.id[:8]} · {s.path.parent.name[-40:]} · {mode}" + (" · PAUSED" if self.paused else "")

        self.query_one("#prompt", Static).update(
            Text.assemble(("last prompt: ", "dim"), s.last_prompt or "—"))
        frac = s.context / limit if limit else 0
        color = "green" if frac < 0.5 else "yellow" if frac < 0.8 else "red"
        self.query_one("#ctx", Static).update(Text.assemble(
            ("Context ", "bold"), (f"{fmt_tok(s.context)} / {fmt_tok(limit)} ({frac:.0%})", f"bold {color}"),
            (f"   peak {fmt_tok(s.peak)} · total output {fmt_tok(s.out_tokens)} · "
             f"compactions {s.compactions} · model {s.model or '?'}   ", "dim"), ("[c] details", "dim italic")))
        barw = max(10, self.query_one("#ctxbar").size.width - 2)
        n = int(round(max(0.0, min(1.0, frac)) * barw))
        self.query_one("#ctxbar", Static).update(
            Text.assemble(("█" * n, color), ("░" * (barw - n), "grey37")))

        running = [a for a in s.agents.values() if not a.finished]
        done = [a for a in s.agents.values() if a.finished]
        self.query_one("#agentsbox").border_title = (
            f"[1] Agents · launched {s.agent_calls} · running {len(running)} · finished {len(done)}"
            " · Enter/f focus · b brief vs result")
        self.query_one("#skillsbox").border_title = (
            f"[2] Skills · available {s.skills_available} · invoked {len(s.skill_calls)}"
            + (f" · new {len(s.skills_added)}" if s.skills_added else ""))
        self.query_one("#feedbox").border_title = "[3] Recent activity · Enter opens details"

        sig = (s.id, s.agent_calls, tuple((a.id, a.finished, a.tools, a.context, a.desc) for a in s.agents.values()),
               len(s.skill_calls), len(s.skills_added), sum(map(len, s.skill_reads.values())),
               len(s.feed), s.feed[-1][:5] if s.feed else None, len(s.results), len(s.main_out), len(s.prompt_ts))
        if sig == self._sig:
            return
        self._sig = sig
        self.rows = {}

        show = sorted(running, key=lambda a: a.start, reverse=True) + sorted(done, key=lambda a: a.start, reverse=True)
        self.fill("agents", [
            (f"ag:{a.id}",
             (Text("● running", style="bold yellow") if not a.finished else Text("✓ done", style="green"),
              Text(a.desc[:40]), Text(a.type, style="dim"), fmt_tok(a.context), str(a.tools),
              verdict_text(s.usage(a), short=True) if a.finished else Text(""), Text(a.last_tool[:40], style="dim"), Text(",".join(dict.fromkeys(a.skills)), style="magenta")),
             ("agent", a)) for a in show])

        rows = []
        for i, call in reversed(list(enumerate(s.skill_calls))):
            ts, who, name, how = call[:4]
            rows.append((f"sc:{i}", (hhmm(ts), Text(name, style="magenta"), Text(how, style="dim"), Text(who, style="dim")),
                         ("skillcall", call)))
        for i, (ts, name) in reversed(list(enumerate(s.skills_added))):
            rows.append((f"sa:{i}", (hhmm(ts), Text("+ " + name, style="cyan"), Text("added", style="dim"), ""),
                         ("skillname", (name, f"added to listing at {hhmm(ts)}"))))
        for k, v in s.skill_reads.items():
            rows.append((f"sr:{k}", ("", Text(k, style="magenta"), Text("read " + ", ".join(sorted(v)), style="dim"), ""),
                         ("skillname", (k, "skill files read"))))
        self.fill("skills", rows)

        rows = []
        for i, entry in enumerate(sorted(s.feed, key=lambda x: x[0], reverse=True)[:200]):
            ts, who, name, detail, tid, _ = entry
            res = s.results.get(tid)
            st = Text("…", style="yellow") if res is None else Text("✗", style="bold red") if res[2] else Text("✓", style="green")
            rows.append((f"fd:{tid or ts + name + str(i)}",
                         (hhmm(ts), Text(who, style="dim" if who == "main" else "cyan"), Text(name, style="bold"),
                          Text(detail[:120]), st), ("feed", entry)))
        self.fill("feed", rows)


def main():
    ap = argparse.ArgumentParser(description="Live X-ray of a Claude Code session.")
    ap.add_argument("--session", help="session id (or path to the .jsonl)")
    ap.add_argument("--all", action="store_true", help="list/follow sessions of every project")
    ap.add_argument("--headless", action="store_true",
                    help="also list/follow headless `claude -p` / SDK runs (hidden by default)")
    ap.add_argument("--cwd", default=os.getcwd(), help="project directory (default: current directory)")
    ap.add_argument("--limit", type=int, default=0, help="context window for the %% bar (default: from the model, 200k or 1M)")
    ap.add_argument("--interval", type=float, default=1.0)
    ap.add_argument("--once", action="store_true", help="print one plain-text snapshot and exit")
    ap.add_argument("--latest", "-l", action="store_true", help="skip the menu and follow the newest session")
    ap.add_argument("--list", type=int, default=15, help="how many sessions the menu shows (default 15)")
    args = ap.parse_args()

    if args.once or not sys.stdout.isatty():
        path = resolve_session(args.session) if args.session else newest_session(args.all, args.cwd, args.headless)
        if not path:
            sys.exit("No interactive session found. Use --all, --headless (claude -p runs) or --session ID.")
        s = Session(path)
        s.update()
        print(render(s, args.limit))
        return
    XRay(args).run()


if __name__ == "__main__":
    main()
