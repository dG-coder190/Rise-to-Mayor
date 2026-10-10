[rise_to_mayor (1).py](https://github.com/user-attachments/files/33291492/rise_to_mayor.1.py)[Uploading#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
RISE TO MAYOR  -  a browser-based strategy game
================================================
Run:   python3 rise_to_mayor.py          (opens http://localhost:8000)
       python3 rise_to_mayor.py 9000     (pick your own port)

Uses only the Python standard library - nothing to install.

The game:
  You are an unknown candidate in the city of Nyhavn with 20 weeks until the
  mayoral election. Each week you have action points (AP) to hold rallies,
  knock on doors, raise money, give interviews, network and run campaigns.
  You pick a political platform (max. 2 issues), hire staff, and handle random
  events, a TV debate and a late scandal. In the end the 5 districts decide
  the election. Your score is saved to the high-score list.
"""
import json
import os
import random
import sys
import threading
import time
import uuid
import webbrowser
from http.cookies import SimpleCookie
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

# --------------------------------------------------------------------------
#  Game data
# --------------------------------------------------------------------------
MAX_WEEKS = 20
BASE_AP = 3
HERE = os.path.dirname(os.path.abspath(__file__))
SCORE_FILE = os.path.join(HERE, "rise_to_mayor_highscores.json")

POLICIES = {
    "jobs": "Jobs",
    "green": "Green transition",
    "safety": "Public safety",
    "housing": "Affordable housing",
    "schools": "Schools & kids",
}

# likes: how much the district likes a policy (-2..2)
DISTRICTS = {
    "harbor": dict(name="Harbor District", pop=18,
                   likes=dict(jobs=2, green=-1, safety=1, housing=1, schools=0)),
    "centrum": dict(name="Downtown", pop=24,
                    likes=dict(jobs=0, green=2, safety=0, housing=1, schools=1)),
    "villa": dict(name="Villa Quarter", pop=20,
                  likes=dict(jobs=0, green=0, safety=2, housing=-2, schools=2)),
    "industry": dict(name="Industrial Zone", pop=16,
                     likes=dict(jobs=2, green=-2, safety=1, housing=1, schools=0)),
    "campus": dict(name="University Quarter", pop=22,
                   likes=dict(jobs=0, green=2, safety=-1, housing=2, schools=1)),
}

ACTIONS = {
    "rally": dict(name="Rally", icon="🎤", ap=1, cost=300, district=True,
                  desc="Speak to a district. Strong effect, costs money."),
    "doors": dict(name="Door-knocking", icon="🚪", ap=1, cost=0, district=True,
                  desc="Free, but slower support gains in one district."),
    "ads": dict(name="TV ad", icon="📺", ap=1, cost=900, district=False,
                desc="A little support citywide + media coverage."),
    "fundraise": dict(name="Fundraising", icon="💰", ap=1, cost=0, district=False,
                      desc="Raise money from small donors. Safe."),
    "donor": dict(name="Donor dinner", icon="🍷", ap=1, cost=0, district=False,
                  desc="Lots of cash - but costs integrity and may end in scandal."),
    "interview": dict(name="Interview", icon="🎙️", ap=1, cost=0, district=False,
                      desc="Boost your media profile. Risk of a gaffe."),
    "network": dict(name="Networking", icon="🤝", ap=1, cost=0, district=False,
                    desc="Build influence, which amplifies all support gains."),
    "attack": dict(name="Attack campaign", icon="🗡️", ap=1, cost=200, district=False,
                   desc="Weakens your rival - but hurts your integrity."),
}

STAFF = {
    "manager": dict(name="Campaign manager", icon="📋", cost=1800,
                    desc="+1 action point every week."),
    "pr": dict(name="Press secretary", icon="📰", cost=1200,
               desc="Fewer gaffes, +1 media every week."),
    "fundraiser": dict(name="Fundraiser", icon="💼", cost=1000,
                       desc="+$250 every week."),
    "analyst": dict(name="Pollster", icon="📊", cost=800,
                    desc="Far more accurate polls."),
    "volunteers": dict(name="Volunteer corps", icon="🧑‍🤝‍🧑", cost=600,
                       desc="Door-knocking +60%, rallies +15%."),
}
WAGE = 100  # weekly wage per employee

ACHIEVEMENTS = {
    "clean": "Clean Conscience – won with integrity ≥ 80",
    "rich": "Moneybags – finished with over $5,000",
    "media": "Media Darling – media ≥ 80 on election day",
    "landslide": "Landslide – over 60% of the vote",
    "comeback": "Comeback – won after polling below 40%",
    "sweep": "Clean Sweep – won all 5 districts",
    "grassroots": "Grassroots – won without hiring a single staffer",
}


def _e(label, fx=None, risk=None, out=None):
    return dict(label=label, fx=fx or {}, risk=risk, out=out)


# risk = (probability, effects on bad luck)
EVENTS = [
    dict(id="flood", title="Flooded basements",
         text="Heavy rain has flooded basements in the Harbor District. Residents are looking for someone to blame.",
         choices=[
             _e("Show up in rubber boots and help (−$200)", dict(money=-200, support_d=dict(harbor=8), media=4)),
             _e("Demand an inquiry into the city sewers", dict(influence=3, support_d=dict(harbor=3), rival=1)),
             _e("Leave it to the city administration", dict(support_d=dict(harbor=-4))),
         ]),
    dict(id="donor", title="A questionable offer",
         text="A local developer offers you $3,000 for your campaign – in exchange for you 'remembering him' later.",
         choices=[
             _e("Take the money", dict(money=3000, integrity=-10), risk=(0.30, dict(media=-12, integrity=-5, support=-2))),
             _e("Politely decline", dict(integrity=3, influence=1)),
         ]),
    dict(id="journalist", title="The reporter calls",
         text="A local reporter is digging into your past and wants a comment.",
         choices=[
             _e("Be completely open", dict(integrity=2, media=6)),
             _e("Give a smooth, rehearsed version", dict(media=10, integrity=-4), risk=(0.35, dict(media=-15, support=-3))),
         ]),
    dict(id="strike", title="Strike at the factory",
         text="Workers at the city's biggest factory have walked out. Everyone is waiting for your reaction.",
         choices=[
             _e("Side with the workers", dict(support_d=dict(industry=9, villa=-3), media=3)),
             _e("Offer to mediate", dict(influence=4, integrity=2, support_d=dict(industry=2))),
             _e("Stay out of it", dict(support_d=dict(industry=-3))),
         ]),
    dict(id="viral", title="Your speech went viral!",
         text="A clip from your latest rally is being shared thousands of times.",
         choices=[
             _e("Seize the moment with a quick call to action", dict(media=12, support=2, money=-100)),
             _e("Stay humble and thank the voters", dict(media=6, integrity=2)),
         ]),
    dict(id="park", title="Fight over City Park",
         text="The city wants to build housing in City Park. Downtown and the University are furious.",
         choices=[
             _e("Fight for the park", dict(support_d=dict(centrum=7, campus=5, harbor=-2), influence=2)),
             _e("Back the housing", dict(support_d=dict(campus=3, centrum=-5, villa=-2), money=300)),
             _e("Propose a referendum", dict(influence=3, integrity=2, media=2)),
         ]),
    dict(id="volunteers", title="Students want to help",
         text="A group of students offers to hand out flyers for you.",
         choices=[
             _e("Accept the help (−$100 for pizza)", dict(money=-100, support_d=dict(campus=6, centrum=3))),
             _e("Decline – you don't want to seem young and unserious", dict(support_d=dict(villa=2))),
         ]),
    dict(id="rumor", title="Rumor about hidden money",
         text="An anonymous online post claims you have hidden income.",
         choices=[
             _e("Refute it publicly with documentation (−$150)", dict(integrity=2, media=3, money=-150)),
             _e("Ignore it", dict(), risk=(0.5, dict(media=-8, integrity=-4))),
         ]),
    dict(id="crime", title="Burglary wave in the Villa Quarter",
         text="Police report a string of break-ins. Neighbors demand action.",
         choices=[
             _e("Demand more officers and cameras", dict(support_d=dict(villa=8, campus=-3))),
             _e("Propose neighborhood watch and better street lighting", dict(support_d=dict(villa=4), integrity=2)),
         ]),
    dict(id="rival_attack", title="Your rival attacks you",
         text="Your opponent calls you 'inexperienced and dangerous for the city's economy' in the newspaper.",
         choices=[
             _e("Hit back in kind", dict(rival=-2, integrity=-4, media=4)),
             _e("Take the high road", dict(integrity=2, media=2, support=-1)),
             _e("Let your press secretary handle it", dict(media=3)),
         ]),
    dict(id="tech", title="Tech company wants to move in",
         text="A major company is considering a new office in the city – if the city is welcoming.",
         choices=[
             _e("Promise tax breaks", dict(influence=3, support_d=dict(industry=4, centrum=3, harbor=2), integrity=-2)),
             _e("Demand green practices and local hiring", dict(integrity=2, support_d=dict(campus=3, centrum=2, industry=2))),
         ]),
    dict(id="gala", title="Charity gala invitation",
         text="The city's business community is hosting a gala. You're invited – tuxedo and smile required.",
         choices=[
             _e("Go and network (−$150)", dict(money=-150, influence=5, media=3)),
             _e("Go and ask for donations", dict(money=900, integrity=-2)),
             _e("Stay home and knock on doors", dict(support=2)),
         ]),
    dict(id="mentor", title="The former mayor calls",
         text="The city's former mayor would like to mentor you over a cup of coffee.",
         choices=[
             _e("Accept the offer", dict(influence=8, media=3)),
             _e("Decline – you want to stand on your own", dict(integrity=3)),
         ]),
    dict(id="bus", title="Bus line to be cut",
         text="The city is cutting route 7. Commuters are angry.",
         choices=[
             _e("Organize a demonstration (−$100)", dict(money=-100, media=5, support=3)),
             _e("Launch a fundraiser for minibuses (−$300)", dict(money=-300, support=4, integrity=2)),
             _e("Blame your rival", dict(rival=-1, integrity=-2)),
         ]),
]

SPECIAL = {
    8: dict(id="profile", title="Big profile in the local paper",
            text="The local paper wants to run a front-page profile of you. How do you want to come across?",
            choices=[
                _e("The down-to-earth family person", dict(media=8, support_d=dict(villa=5, industry=3))),
                _e("The visionary with big plans", dict(media=8, support_d=dict(campus=5, centrum=4))),
                _e("The tough negotiator", dict(media=6, influence=5, integrity=-2)),
            ]),
    14: dict(id="debate", title="📺 The Mayoral Debate",
             text="The whole city is watching. Your opponent stands beside you. Choose your strategy.",
             choices=[
                 _e("Go on the attack", dict(media=15, support=3, rival=-3), risk=(0.5, dict(media=-22, integrity=-3, support=-4))),
                 _e("Present facts and concrete plans", dict(support=2, integrity=2, media=5, influence=3)),
                 _e("Use charm and humor", dict(media=10, support=1), risk=(0.3, dict(support=-4, media=-5))),
             ]),
    18: dict(id="october", title="🚨 The October Surprise",
             text="An old, compromising photo of you surfaces two weeks before the election.",
             choices=[
                 _e("Be honest and apologize", dict(integrity=3, media=-4)),
                 _e("Deny everything", dict(), risk=(0.5, dict(integrity=-20, media=-15, support=-4))),
                 _e("Counter-attack your rival", dict(rival=-4, integrity=-6, media=2)),
             ]),
}

# --------------------------------------------------------------------------
#  Game logic
# --------------------------------------------------------------------------


class GameError(Exception):
    pass


def clamp(v, lo, hi):
    return max(lo, min(hi, v))


class Game:
    def __init__(self, name, difficulty="normal"):
        self.rng = random.Random()
        self.name = (name or "The Candidate").strip()[:24] or "The Candidate"
        self.difficulty = difficulty if difficulty in ("easy", "normal", "hard") else "normal"
        self.rival_mult = dict(easy=0.8, normal=1.0, hard=1.25)[self.difficulty]
        self.week = 1
        self.ap = BASE_AP
        self.money = 1500
        self.integrity = 70
        self.media = 20
        self.influence = 10
        self.platform = []
        self.staff = []
        self.event = None
        self.over = False
        self.result = None
        self.log = []
        self.min_poll = 100.0
        self.hired_ever = False
        self.support = {d: self.rng.uniform(13, 21) for d in DISTRICTS}
        self.rival = {d: self.rng.uniform(27, 33) for d in DISTRICTS}
        self.noise = {}
        self._new_noise()
        self.add_log("You announce your candidacy for mayor of Nyhavn. 20 weeks until the election!")

    # ----- helpers
    def add_log(self, text):
        self.log.append(dict(week=self.week, text=text))
        self.log = self.log[-40:]

    def _new_noise(self):
        sd = 2 if "analyst" in self.staff else 7
        self.noise = {d: self.rng.gauss(0, sd) for d in DISTRICTS}

    def appeal(self, d):
        return sum(DISTRICTS[d]["likes"][p] * 3 for p in self.platform)

    def share(self, d, noise=0.0):
        p = (self.support[d] + self.appeal(d) + (self.integrity - 50) * 0.15
             + (self.media - 30) * 0.10)
        p = max(1.0, p)
        r = max(1.0, self.rival[d])
        return clamp(p / (p + r) * 100 + noise, 1, 99)

    def total_share(self, noisy=False):
        tot = sum(v["pop"] for v in DISTRICTS.values())
        return sum(self.share(d, self.noise[d] if noisy else 0) * DISTRICTS[d]["pop"]
                   for d in DISTRICTS) / tot

    def max_ap(self):
        return BASE_AP + (1 if "manager" in self.staff else 0)

    def gain_mult(self):
        return 1 + self.influence / 150.0

    def snap(self):
        return dict(money=self.money, integrity=self.integrity, media=self.media,
                    influence=self.influence, poll=self.total_share())

    def diff(self, before):
        after = self.snap()
        out = []
        names = dict(money=("Money", "$"), integrity=("Integrity", ""),
                     media=("Media", ""), influence=("Influence", ""))
        for k, (label, unit) in names.items():
            d = round(after[k] - before[k])
            if d:
                out.append(dict(label=label, value=d, unit=unit))
        dp = round(after["poll"] - before["poll"], 1)
        if dp:
            out.append(dict(label="Poll", value=dp, unit=" pts"))
        return out

    def apply(self, fx):
        for k, v in fx.items():
            if k == "money":
                self.money += v
            elif k == "integrity":
                self.integrity = clamp(self.integrity + v, 0, 100)
            elif k == "media":
                self.media = clamp(self.media + v, 0, 100)
            elif k == "influence":
                self.influence = max(0, self.influence + v)
            elif k == "support":
                for d in DISTRICTS:
                    self.support[d] = max(0, self.support[d] + v * self.gain_mult() if v > 0 else self.support[d] + v)
            elif k == "support_d":
                for d, val in v.items():
                    self.support[d] = max(0, self.support[d] + (val * self.gain_mult() if val > 0 else val))
            elif k == "rival":
                for d in DISTRICTS:
                    self.rival[d] = max(5, self.rival[d] + v)

    def _require_active(self):
        if self.over:
            raise GameError("The game is over. Start a new game.")
        if self.event:
            raise GameError("You must respond to the event first.")

    # ----- actions
    def do_action(self, key, district=None):
        self._require_active()
        if key not in ACTIONS:
            raise GameError("Unknown action.")
        a = ACTIONS[key]
        if self.ap < a["ap"]:
            raise GameError("No action points left this week – end the week.")
        if self.money < a["cost"]:
            raise GameError("You can't afford that.")
        if a["district"] and district not in DISTRICTS:
            raise GameError("Pick a district.")
        before = self.snap()
        r = self.rng
        vol = "volunteers" in self.staff
        gm = self.gain_mult()
        self.ap -= a["ap"]
        self.money -= a["cost"]
        msg = ""

        if key == "rally":
            g = r.uniform(6, 9.5) * (1.15 if vol else 1) * gm
            self.support[district] += g
            self.media = clamp(self.media + 2, 0, 100)
            msg = f"The rally in {DISTRICTS[district]['name']} was well attended."
        elif key == "doors":
            g = r.uniform(3, 5) * (1.6 if vol else 1) * gm
            self.support[district] += g
            msg = f"You knocked on doors in {DISTRICTS[district]['name']} and talked with people."
        elif key == "ads":
            for d in DISTRICTS:
                self.support[d] += r.uniform(2.2, 3.4) * gm
            self.media = clamp(self.media + 5, 0, 100)
            msg = "The ad ran in prime time."
        elif key == "fundraise":
            got = int(r.uniform(450, 800))
            self.money += got
            msg = f"You raised ${got} from small donors."
        elif key == "donor":
            got = int(r.uniform(1300, 1900))
            self.money += got
            self.integrity = clamp(self.integrity - 6, 0, 100)
            msg = f"The donor dinner raised ${got}."
            if r.random() < 0.22:
                self.media = clamp(self.media - 10, 0, 100)
                self.integrity = clamp(self.integrity - 4, 0, 100)
                msg += " But a reporter caught wind of it – scandal!"
        elif key == "interview":
            p_gaffe = 0.08 if "pr" in self.staff else 0.25
            if r.random() < p_gaffe:
                self.media = clamp(self.media - 8, 0, 100)
                for d in DISTRICTS:
                    self.support[d] = max(0, self.support[d] - 2)
                msg = "You said something unfortunate on air. Gaffe!"
            else:
                self.media = clamp(self.media + r.uniform(8, 13), 0, 100)
                for d in DISTRICTS:
                    self.support[d] += 1 * gm
                msg = "The interview went really well."
        elif key == "network":
            self.influence += 5
            msg = "You made valuable connections."
            if r.random() < 0.30:
                self.money += 300
                msg += " One of them made a small donation (+$300)."
        elif key == "attack":
            self.apply(dict(rival=-3, integrity=-6))
            msg = "You released a sharp attack on your rival's past."
            if r.random() < 0.20:
                self.apply(dict(media=-6))
                msg += " The press called it 'dirty' – it backfires."
        self.add_log(msg)
        self.min_poll = min(self.min_poll, self.total_share())
        return msg, self.diff(before)

    def hire(self, key):
        self._require_active()
        if key not in STAFF:
            raise GameError("Unknown staff member.")
        if key in self.staff:
            raise GameError("You have already hired this person.")
        s = STAFF[key]
        if self.money < s["cost"]:
            raise GameError("You can't afford that.")
        before = self.snap()
        self.money -= s["cost"]
        self.staff.append(key)
        self.hired_ever = True
        if key == "manager":
            self.ap += 1
        if key == "analyst":
            self._new_noise()
        msg = f"You hired a {s['name']}."
        self.add_log(msg)
        return msg, self.diff(before)

    def set_platform(self, policies):
        self._require_active()
        policies = [p for p in policies if p in POLICIES][:2]
        before = self.snap()
        removed = [p for p in self.platform if p not in policies]
        msg = "Platform updated."
        if self.week > 1 and removed and self.platform:
            self.integrity = clamp(self.integrity - 3 * len(removed), 0, 100)
            msg = "You changed your position – voters call it flip-flopping."
        self.platform = policies
        names = ", ".join(POLICIES[p] for p in policies) or "none"
        self.add_log(f"Platform: {names}.")
        self.min_poll = min(self.min_poll, self.total_share())
        return msg, self.diff(before)

    # ----- events
    def _draw_event(self):
        if self.week in SPECIAL:
            ev = SPECIAL[self.week]
        elif self.rng.random() < 0.65:
            used = {l.get("event") for l in self.log}
            pool = [e for e in EVENTS if e["id"] not in used] or EVENTS
            ev = self.rng.choice(pool)
        else:
            return
        self.event = ev
        self.log.append(dict(week=self.week, text=f"Event: {ev['title']}", event=ev["id"]))

    def choose(self, idx):
        if self.over or not self.event:
            raise GameError("There is no active event.")
        try:
            c = self.event["choices"][int(idx)]
        except (ValueError, IndexError, TypeError):
            raise GameError("Invalid choice.")
        if c["fx"].get("money", 0) < 0 and self.money + c["fx"]["money"] < 0:
            raise GameError("You can't afford that choice.")
        before = self.snap()
        self.apply(c["fx"])
        text = f"{self.event['title']}: {c['label']}"
        bad = False
        if c["risk"] and self.rng.random() < c["risk"][0]:
            self.apply(c["risk"][1])
            bad = True
            text += " – it backfired!"
        self.money = max(0, self.money)
        self.add_log(text)
        self.event = None
        self.min_poll = min(self.min_poll, self.total_share())
        return ("It backfired! " if bad else "") + "Decision made.", self.diff(before)

    # ----- week change
    def end_week(self):
        self._require_active()
        before = self.snap()
        r = self.rng
        # income and expenses
        income = 250 if "fundraiser" in self.staff else 0
        wages = WAGE * len(self.staff)
        self.money = max(0, self.money + income - wages)
        if "pr" in self.staff:
            self.media = clamp(self.media + 1, 0, 100)
        # support and media fade
        for d in DISTRICTS:
            self.support[d] = max(0, self.support[d] - 0.8)
        self.media = clamp(self.media - 1, 0, 100)
        # the rival grows - and focuses on your strongest district
        best = max(DISTRICTS, key=lambda d: self.share(d))
        for d in DISTRICTS:
            self.rival[d] += r.uniform(1.4, 2.8) * self.rival_mult
        self.rival[best] += 1.5 * self.rival_mult
        self.add_log(f"Week {self.week} ended. Wages: −${wages}, income: +${income}.")
        self.week += 1
        if self.week > MAX_WEEKS:
            self._election()
            return "Election day has arrived!", self.diff(before)
        self.ap = self.max_ap()
        self._new_noise()
        self._draw_event()
        self.min_poll = min(self.min_poll, self.total_share())
        return f"Week {self.week} begins.", self.diff(before)

    # ----- election
    def _election(self):
        r = self.rng
        rows = []
        votes_me = votes_rival = 0.0
        for d, info in DISTRICTS.items():
            turnout = r.uniform(0.55, 0.78)
            s = clamp(self.share(d) + r.gauss(0, 1.5), 1, 99)
            tot = info["pop"] * 1000 * turnout
            votes_me += tot * s / 100
            votes_rival += tot * (100 - s) / 100
            rows.append(dict(id=d, name=info["name"], share=round(s, 1), won=s > 50,
                             votes=int(tot * s / 100)))
        total = votes_me / (votes_me + votes_rival) * 100
        win = total > 50
        won_districts = sum(1 for x in rows if x["won"])
        ach = []
        if win and self.integrity >= 80:
            ach.append("clean")
        if self.money >= 5000:
            ach.append("rich")
        if self.media >= 80:
            ach.append("media")
        if total >= 60:
            ach.append("landslide")
        if win and self.min_poll < 40:
            ach.append("comeback")
        if won_districts == len(DISTRICTS):
            ach.append("sweep")
        if win and not self.hired_ever:
            ach.append("grassroots")
        mult = dict(easy=0.8, normal=1.0, hard=1.3)[self.difficulty]
        score = int((total * 20 + self.integrity * 3 + self.money / 10
                     + (1000 if win else 0) + len(ach) * 150) * mult)
        self.result = dict(win=win, total=round(total, 1), districts=rows, score=score,
                           integrity=self.integrity, money=self.money,
                           achievements=[dict(id=a, text=ACHIEVEMENTS[a]) for a in ach])
        self.over = True
        self.event = None
        self.add_log("The election is decided: " + ("YOU ARE THE MAYOR!" if win else "You lost the election."))
        save_score(self.name, score, total, win, self.difficulty)

    # ----- serialization
    def to_dict(self):
        title = ("Unknown candidate", "Local hope", "Grassroots challenger", "Frontrunner for mayor")
        poll = self.total_share(noisy=not self.over)
        rank = title[0 if poll < 35 else 1 if poll < 43 else 2 if poll < 50 else 3]
        districts = []
        for d, info in DISTRICTS.items():
            districts.append(dict(
                id=d, name=info["name"], pop=info["pop"], likes=info["likes"],
                poll=round(self.share(d, self.noise[d] if not self.over else 0), 1),
                appeal=self.appeal(d)))
        ev = None
        if self.event:
            ev = dict(title=self.event["title"], text=self.event["text"],
                      choices=[c["label"] for c in self.event["choices"]])
        return dict(
            name=self.name, difficulty=self.difficulty, week=min(self.week, MAX_WEEKS),
            max_weeks=MAX_WEEKS, ap=self.ap, max_ap=self.max_ap(), money=int(self.money),
            integrity=round(self.integrity), media=round(self.media),
            influence=round(self.influence), platform=self.platform, staff=self.staff,
            poll=round(poll, 1), rank=rank, districts=districts, event=ev,
            log=self.log[-14:][::-1], over=self.over, result=self.result,
            actions=ACTIONS, policies=POLICIES, staff_defs=STAFF, wage=WAGE,
            precise=("analyst" in self.staff))


# --------------------------------------------------------------------------
#  High scores
# --------------------------------------------------------------------------
_score_lock = threading.Lock()


def load_scores():
    try:
        with open(SCORE_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except (OSError, ValueError):
        return []


def save_score(name, score, share, win, difficulty):
    with _score_lock:
        scores = load_scores()
        scores.append(dict(name=name, score=score, share=round(share, 1), win=win,
                           difficulty=difficulty, date=time.strftime("%Y-%m-%d")))
        scores.sort(key=lambda s: s["score"], reverse=True)
        try:
            with open(SCORE_FILE, "w", encoding="utf-8") as f:
                json.dump(scores[:20], f, ensure_ascii=False, indent=1)
        except OSError:
            pass


# --------------------------------------------------------------------------
#  Web server
# --------------------------------------------------------------------------
SESSIONS = {}
SESS_LOCK = threading.Lock()


class Handler(BaseHTTPRequestHandler):
    server_version = "RiseToMayor/1.0"

    def log_message(self, fmt, *args):
        pass

    def _sid(self):
        c = SimpleCookie(self.headers.get("Cookie", ""))
        if "sid" in c and c["sid"].value:
            return c["sid"].value, False
        return uuid.uuid4().hex, True

    def _send(self, code, body, ctype="application/json; charset=utf-8", sid=None, new=False):
        data = body if isinstance(body, bytes) else body.encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(data)))
        self.send_header("Cache-Control", "no-store")
        if new and sid:
            self.send_header("Set-Cookie", f"sid={sid}; Path=/; HttpOnly; SameSite=Lax")
        self.end_headers()
        self.wfile.write(data)

    def _json(self, obj, code=200, sid=None, new=False):
        self._send(code, json.dumps(obj, ensure_ascii=False), sid=sid, new=new)

    def do_GET(self):
        sid, new = self._sid()
        path = self.path.split("?")[0]
        if path == "/":
            self._send(200, HTML, "text/html; charset=utf-8", sid, new)
        elif path == "/api/state":
            g = SESSIONS.get(sid)
            self._json(dict(game=g.to_dict() if g else None, scores=load_scores()[:10]), sid=sid, new=new)
        elif path == "/api/scores":
            self._json(load_scores()[:10], sid=sid, new=new)
        else:
            self._send(404, "Not found", "text/plain; charset=utf-8")

    def do_POST(self):
        sid, new = self._sid()
        try:
            n = int(self.headers.get("Content-Length", 0) or 0)
            body = json.loads(self.rfile.read(n).decode("utf-8") or "{}") if n else {}
        except ValueError:
            return self._json(dict(error="Invalid request."), 400, sid, new)
        path = self.path.split("?")[0]
        try:
            with SESS_LOCK:
                if path == "/api/new":
                    SESSIONS[sid] = Game(body.get("name"), body.get("difficulty", "normal"))
                    out = dict(msg="The campaign is underway!", changes=[])
                else:
                    g = SESSIONS.get(sid)
                    if not g:
                        raise GameError("No active game – start a new one.")
                    if path == "/api/action":
                        msg, ch = g.do_action(body.get("action"), body.get("district"))
                    elif path == "/api/hire":
                        msg, ch = g.hire(body.get("staff"))
                    elif path == "/api/platform":
                        msg, ch = g.set_platform(body.get("policies") or [])
                    elif path == "/api/event":
                        msg, ch = g.choose(body.get("choice"))
                    elif path == "/api/endweek":
                        msg, ch = g.end_week()
                    else:
                        return self._send(404, "Not found", "text/plain; charset=utf-8")
                    out = dict(msg=msg, changes=ch)
                g = SESSIONS[sid]
                out["game"] = g.to_dict()
                out["scores"] = load_scores()[:10]
            self._json(out, sid=sid, new=new)
        except GameError as e:
            self._json(dict(error=str(e)), 400, sid, new)


# --------------------------------------------------------------------------
#  Frontend (HTML + CSS + JS)
# --------------------------------------------------------------------------
HTML = r"""<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Rise to Mayor</title>
<style>
:root{--bg:#0e1424;--panel:#161e36;--panel2:#1d2745;--line:#2b3760;--txt:#e8ecf8;--mut:#93a0c4;
--gold:#f5c542;--good:#3ecf8e;--bad:#ff6b6b;--warn:#ffb347;--blue:#5b8cff}
*{box-sizing:border-box}
body{margin:0;font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;background:
radial-gradient(1200px 600px at 80% -10%,#1d2a5a 0,transparent 60%),var(--bg);color:var(--txt);min-height:100vh}
h1,h2,h3{margin:0}
button{font:inherit;cursor:pointer;border:0;color:inherit}
.wrap{max-width:1280px;margin:0 auto;padding:16px}
header{display:flex;flex-wrap:wrap;gap:14px;align-items:center;justify-content:space-between;margin-bottom:14px}
.brand{display:flex;align-items:center;gap:10px}
.brand .logo{font-size:30px}
.brand h1{font-size:22px;letter-spacing:.5px}
.brand small{color:var(--mut);display:block;font-size:12px}
.weekbox{flex:1;min-width:240px;max-width:460px}
.weekbox .lbl{display:flex;justify-content:space-between;font-size:13px;color:var(--mut);margin-bottom:4px}
.bar{height:10px;background:#0b1020;border-radius:99px;overflow:hidden;border:1px solid var(--line)}
.bar>i{display:block;height:100%;background:linear-gradient(90deg,var(--blue),var(--gold));transition:width .5s}
.ap{display:flex;gap:6px;align-items:center}
.pip{width:18px;height:18px;border-radius:50%;border:2px solid var(--gold);background:transparent}
.pip.on{background:var(--gold);box-shadow:0 0 10px #f5c54288}
.stats{display:grid;grid-template-columns:repeat(auto-fit,minmax(150px,1fr));gap:10px;margin-bottom:14px}
.stat{background:var(--panel);border:1px solid var(--line);border-radius:14px;padding:10px 14px}
.stat .k{color:var(--mut);font-size:12px;text-transform:uppercase;letter-spacing:.8px}
.stat .v{font-size:24px;font-weight:700;margin-top:2px}
.stat .mini{height:5px;background:#0b1020;border-radius:9px;margin-top:6px;overflow:hidden}
.stat .mini i{display:block;height:100%;transition:width .5s}
.grid{display:grid;grid-template-columns:1.1fr 1.2fr 1fr;gap:14px}
@media(max-width:1000px){.grid{grid-template-columns:1fr}}
.card{background:var(--panel);border:1px solid var(--line);border-radius:16px;padding:14px}
.card h2{font-size:15px;color:var(--gold);text-transform:uppercase;letter-spacing:1px;margin-bottom:10px}
.poll{text-align:center;margin-bottom:10px}
.poll .big{font-size:44px;font-weight:800}
.poll .rank{color:var(--mut);font-size:13px}
.dist{margin-bottom:12px}
.dist .row{display:flex;justify-content:space-between;font-size:14px;margin-bottom:3px}
.dist .row b{font-weight:600}
.track{position:relative;height:14px;background:#0b1020;border-radius:8px;border:1px solid var(--line);overflow:hidden}
.track i{display:block;height:100%;transition:width .5s}
.track::after{content:"";position:absolute;left:50%;top:0;bottom:0;width:2px;background:#ffffff66}
.chips{display:flex;flex-wrap:wrap;gap:4px;margin-top:4px}
.chip{font-size:11px;padding:2px 7px;border-radius:99px;background:var(--panel2);color:var(--mut);border:1px solid var(--line)}
.chip.p{color:var(--good);border-color:#3ecf8e55}.chip.n{color:var(--bad);border-color:#ff6b6b55}
.acts{display:grid;grid-template-columns:1fr 1fr;gap:10px}
.act{background:var(--panel2);border:1px solid var(--line);border-radius:12px;padding:10px;text-align:left;transition:.15s}
.act:hover:not(:disabled){transform:translateY(-2px);border-color:var(--gold)}
.act:disabled{opacity:.4;cursor:not-allowed}
.act .t{font-weight:700;font-size:14px}
.act .d{font-size:12px;color:var(--mut);margin:3px 0 6px;min-height:30px}
.act .c{font-size:12px;color:var(--gold)}
.end{width:100%;margin-top:12px;padding:14px;border-radius:12px;background:linear-gradient(90deg,#ffb347,#f5c542);color:#1a1300;font-weight:800;font-size:16px}
.end:hover{filter:brightness(1.08)}
.pol{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:6px}
.pol button{padding:8px 12px;border-radius:99px;background:var(--panel2);border:1px solid var(--line)}
.pol button.on{background:var(--blue);border-color:var(--blue);font-weight:700}
.hint{font-size:12px;color:var(--mut);margin-bottom:12px}
.staff{display:grid;gap:8px;margin-bottom:12px}
.st{display:flex;align-items:center;justify-content:space-between;gap:8px;background:var(--panel2);border:1px solid var(--line);border-radius:10px;padding:8px 10px}
.st .n{font-size:13px;font-weight:600}.st .d{font-size:11px;color:var(--mut)}
.st button{background:var(--blue);padding:6px 10px;border-radius:8px;font-size:12px;white-space:nowrap}
.st button:disabled{background:#2b3760;opacity:.6;cursor:not-allowed}
.st .own{color:var(--good);font-size:12px;font-weight:700}
.log{max-height:230px;overflow:auto;font-size:13px}
.log div{padding:6px 0;border-bottom:1px dashed var(--line);color:var(--mut)}
.log b{color:var(--gold);margin-right:6px}
.overlay{position:fixed;inset:0;background:#050810d0;display:none;align-items:center;justify-content:center;padding:16px;z-index:50;backdrop-filter:blur(3px)}
.overlay.show{display:flex}
.modal{background:var(--panel);border:1px solid var(--gold);border-radius:18px;padding:22px;max-width:560px;width:100%;max-height:92vh;overflow:auto;animation:pop .25s}
@keyframes pop{from{transform:scale(.92);opacity:0}to{transform:none;opacity:1}}
.modal h2{font-size:22px;margin-bottom:8px}
.modal p{color:var(--mut);line-height:1.5}
.choice{display:block;width:100%;text-align:left;margin-top:10px;padding:12px 14px;border-radius:12px;background:var(--panel2);border:1px solid var(--line)}
.choice:hover{border-color:var(--gold);background:#243055}
input[type=text]{width:100%;padding:12px;border-radius:10px;border:1px solid var(--line);background:#0b1020;color:var(--txt);font:inherit;margin:8px 0}
.diff{display:flex;gap:8px;margin:10px 0}
.diff button{flex:1;padding:10px;border-radius:10px;background:var(--panel2);border:1px solid var(--line)}
.diff button.on{border-color:var(--gold);background:#2a3560;font-weight:700}
.primary{width:100%;margin-top:10px;padding:13px;border-radius:12px;background:var(--gold);color:#1a1300;font-weight:800;font-size:16px}
.ghost{width:100%;margin-top:8px;padding:10px;border-radius:12px;background:transparent;border:1px solid var(--line);color:var(--mut)}
.hs{width:100%;border-collapse:collapse;font-size:13px;margin-top:8px}
.hs td,.hs th{padding:5px 6px;border-bottom:1px solid var(--line);text-align:left}
.hs th{color:var(--mut);font-weight:500}
.res-big{font-size:54px;font-weight:900;text-align:center;margin:6px 0}
.win{color:var(--good)}.lose{color:var(--bad)}
.ach{padding:6px 10px;margin:4px 0;border-radius:8px;background:#f5c54222;border:1px solid #f5c54255;font-size:13px}
#toast{position:fixed;left:50%;bottom:20px;transform:translateX(-50%);z-index:80;display:flex;flex-direction:column;gap:8px;align-items:center;pointer-events:none}
.tt{background:#0b1020f2;border:1px solid var(--gold);border-radius:12px;padding:10px 16px;max-width:90vw;animation:pop .2s;font-size:14px}
.tt.err{border-color:var(--bad);color:#ffb3b3}
.tt .chg{display:flex;flex-wrap:wrap;gap:10px;margin-top:5px;font-size:12px}
.up{color:var(--good)}.dn{color:var(--bad)}
.badge{font-size:11px;background:#2b3760;border-radius:99px;padding:2px 8px;color:var(--mut);margin-left:6px}
</style>
</head>
<body>
<div class="wrap" id="app">
  <header>
    <div class="brand"><div class="logo">🏛️</div>
      <div><h1>RISE TO MAYOR</h1><small id="sub">Your road to the mayor's office</small></div></div>
    <div class="weekbox"><div class="lbl"><span id="weekLbl">Week 1 / 20</span><span id="timeLeft"></span></div>
      <div class="bar"><i id="weekBar" style="width:0"></i></div></div>
    <div class="ap" title="Action points this week"><span style="color:var(--mut);font-size:13px">Action points</span><span id="pips" class="ap"></span></div>
  </header>

  <div class="stats" id="stats"></div>

  <div class="grid">
    <section class="card">
      <h2>📍 The Polls</h2>
      <div class="poll"><div class="big" id="pollBig">–</div><div class="rank" id="rank"></div>
        <div class="rank" id="precise"></div></div>
      <div id="districts"></div>
    </section>

    <section class="card">
      <h2>⚡ Actions</h2>
      <div class="acts" id="acts"></div>
      <button class="end" id="endBtn">End week ➜</button>
    </section>

    <section class="card">
      <h2>📜 Platform (max. 2)</h2>
      <div class="pol" id="policies"></div>
      <div class="hint">Each district likes or dislikes different issues. Changing your mind costs integrity.</div>
      <h2>👥 Staff</h2>
      <div class="staff" id="staff"></div>
      <h2>🗒️ Campaign log</h2>
      <div class="log" id="log"></div>
    </section>
  </div>
</div>

<div class="overlay" id="ovl"><div class="modal" id="modal"></div></div>
<div id="toast"></div>

<script>
let G=null, SCORES=[], pickedDiff="normal", busy=false;
const $=id=>document.getElementById(id);
const fmt=n=>n.toLocaleString("en-US");

async function api(path, body){
  if(busy) return null; busy=true;
  try{
    const r=await fetch(path,{method:body===undefined?"GET":"POST",headers:{"Content-Type":"application/json"},body:body===undefined?undefined:JSON.stringify(body)});
    const j=await r.json();
    if(!r.ok){toast(j.error||"Error",null,true);return null}
    if(j.game){G=j.game} if(j.scores){SCORES=j.scores}
    return j;
  }catch(e){toast("Could not reach the server",null,true);return null}
  finally{busy=false}
}
function toast(msg,changes,err){
  const t=document.createElement("div");t.className="tt"+(err?" err":"");
  let h=`<div>${msg}</div>`;
  if(changes&&changes.length){h+='<div class="chg">'+changes.map(c=>`<span class="${c.value>0?"up":"dn"}">${c.value>0?"▲":"▼"} ${c.label} ${c.value>0?"+":""}${fmt(c.value)}${c.unit}</span>`).join("")+"</div>"}
  t.innerHTML=h;$("toast").appendChild(t);setTimeout(()=>t.remove(),err?3200:4200);
}
function colorFor(p){return p>=52?"var(--good)":p>=44?"var(--warn)":"var(--bad)"}

function render(){
  if(!G) return;
  $("weekLbl").textContent=`Week ${G.week} / ${G.max_weeks}`;
  $("timeLeft").textContent=G.over?"Election decided":`${G.max_weeks-G.week} weeks to the election`;
  $("weekBar").style.width=((G.week-1)/G.max_weeks*100)+"%";
  $("sub").textContent=`${G.name} · ${({easy:"Easy",normal:"Normal",hard:"Hard"})[G.difficulty]}`;
  $("pips").innerHTML=Array.from({length:G.max_ap},(_,i)=>`<span class="pip ${i<G.ap?"on":""}"></span>`).join("");
  const bar=(v,c)=>`<div class="mini"><i style="width:${v}%;background:${c}"></i></div>`;
  $("stats").innerHTML=
   `<div class="stat"><div class="k">💰 Money</div><div class="v">$${fmt(G.money)}</div></div>`+
   `<div class="stat"><div class="k">⚖️ Integrity</div><div class="v">${G.integrity}</div>${bar(G.integrity,"var(--good)")}</div>`+
   `<div class="stat"><div class="k">📣 Media</div><div class="v">${G.media}</div>${bar(G.media,"var(--blue)")}</div>`+
   `<div class="stat"><div class="k">🧠 Influence</div><div class="v">${G.influence}</div><div class="k" style="margin-top:4px;text-transform:none">+${Math.round(G.influence/1.5)} % to all support gains</div></div>`;
  $("pollBig").textContent=G.poll.toFixed(1)+" %";
  $("pollBig").style.color=colorFor(G.poll);
  $("rank").textContent=G.rank;
  $("precise").textContent=G.precise?"Accurate poll (pollster hired)":"Poll is uncertain (±7) – hire a pollster";
  $("districts").innerHTML=G.districts.map(d=>{
    const chips=Object.entries(d.likes).filter(([k,v])=>v!==0).map(([k,v])=>`<span class="chip ${v>0?"p":"n"}">${v>0?"♥":"✖"} ${G.policies[k]}</span>`).join("");
    const ap=d.appeal?` <span class="badge">platform ${d.appeal>0?"+":""}${d.appeal}</span>`:"";
    return `<div class="dist"><div class="row"><b>${d.name}${ap}</b><span style="color:${colorFor(d.poll)}">${d.poll.toFixed(0)} %</span></div>
    <div class="track"><i style="width:${d.poll}%;background:${colorFor(d.poll)}"></i></div><div class="chips">${chips}</div></div>`}).join("");
  $("acts").innerHTML=Object.entries(G.actions).map(([k,a])=>{
    const dis=G.over||G.event||G.ap<a.ap||G.money<a.cost;
    return `<button class="act" ${dis?"disabled":""} data-k="${k}"><div class="t">${a.icon} ${a.name}</div><div class="d">${a.desc}</div><div class="c">${a.ap} AP${a.cost?` · $${a.cost}`:""}</div></button>`}).join("");
  document.querySelectorAll(".act").forEach(b=>b.onclick=()=>onAction(b.dataset.k));
  $("endBtn").disabled=!!(G.over||G.event);
  $("endBtn").textContent=G.week>=G.max_weeks?"Go to the election 🗳️":"End week ➜";
  $("policies").innerHTML=Object.entries(G.policies).map(([k,n])=>`<button class="${G.platform.includes(k)?"on":""}" data-p="${k}">${n}</button>`).join("");
  document.querySelectorAll("[data-p]").forEach(b=>b.onclick=()=>togglePolicy(b.dataset.p));
  $("staff").innerHTML=Object.entries(G.staff_defs).map(([k,s])=>{
    const own=G.staff.includes(k);
    return `<div class="st"><div><div class="n">${s.icon} ${s.name}</div><div class="d">${s.desc}</div></div>${own?'<span class="own">Hired ✔</span>':`<button data-h="${k}" ${G.money<s.cost||G.over||G.event?"disabled":""}>Hire $${fmt(s.cost)}</button>`}</div>`}).join("")+
    `<div class="hint">Wages: $${G.wage} per employee per week.</div>`;
  document.querySelectorAll("[data-h]").forEach(b=>b.onclick=()=>hire(b.dataset.h));
  $("log").innerHTML=G.log.map(l=>`<div><b>W${l.week}</b>${l.text}</div>`).join("");
  showModal();
}

function onAction(k){
  const a=G.actions[k];
  if(!a.district) return doAction(k);
  openModal(`<h2>${a.icon} ${a.name}</h2><p>Pick a district:</p>`+G.districts.map(d=>`<button class="choice" data-d="${d.id}">${d.name} <span class="badge">${d.poll.toFixed(0)} %</span></button>`).join("")+`<button class="ghost" id="cancel">Cancel</button>`,true);
  document.querySelectorAll("[data-d]").forEach(b=>b.onclick=()=>{closeModal();doAction(k,b.dataset.d)});
  $("cancel").onclick=closeModal;
}
async function doAction(k,d){const j=await api("/api/action",{action:k,district:d});if(j){toast(j.msg,j.changes);render()}}
async function hire(k){const j=await api("/api/hire",{staff:k});if(j){toast(j.msg,j.changes);render()}}
async function togglePolicy(p){
  let l=[...G.platform];
  if(l.includes(p)) l=l.filter(x=>x!==p); else{ if(l.length>=2) l.shift(); l.push(p)}
  const j=await api("/api/platform",{policies:l});if(j){toast(j.msg,j.changes);render()}
}
$("endBtn").onclick=async()=>{const j=await api("/api/endweek",{});if(j){toast(j.msg,j.changes);render()}};

// ---- modals
let modalSticky=false;
function openModal(html,closable){$("modal").innerHTML=html;$("ovl").classList.add("show");modalSticky=!!closable}
function closeModal(){$("ovl").classList.remove("show");modalSticky=false}
$("ovl").onclick=e=>{if(e.target.id==="ovl"&&modalSticky)closeModal()};

function showModal(){
  if(!G) return;
  if(G.over&&G.result) return showResult();
  if(G.event){
    openModal(`<h2>${G.event.title}</h2><p>${G.event.text}</p>`+G.event.choices.map((c,i)=>`<button class="choice" data-c="${i}">${c}</button>`).join(""));
    document.querySelectorAll("[data-c]").forEach(b=>b.onclick=async()=>{const j=await api("/api/event",{choice:b.dataset.c});if(j){closeModal();toast(j.msg,j.changes);render()}});
  } else if(!modalSticky) closeModal();
}
function scoreTable(){
  if(!SCORES.length) return "<p>No scores yet.</p>";
  return `<table class="hs"><tr><th>#</th><th>Name</th><th>Score</th><th>Votes</th><th></th></tr>`+SCORES.map((s,i)=>`<tr><td>${i+1}</td><td>${s.name.replace(/</g,"&lt;")}</td><td>${fmt(s.score)}</td><td>${s.share} %</td><td>${s.win?"🏆":""}</td></tr>`).join("")+"</table>";
}
function showResult(){
  const r=G.result;
  openModal(`<div class="res-big ${r.win?"win":"lose"}">${r.win?"🏆 MAYOR!":"Election lost"}</div>
  <p style="text-align:center">${r.win?"The city has spoken – you're moving into city hall.":"Not this time. Try a different strategy."}</p>
  <h3 style="text-align:center;margin:10px 0">${r.total} % of the vote · Score ${fmt(r.score)}</h3>`+
  r.districts.map(d=>`<div class="dist"><div class="row"><b>${d.name}</b><span style="color:${colorFor(d.share)}">${d.share} % ${d.won?"✔":"✖"}</span></div><div class="track"><i style="width:${d.share}%;background:${colorFor(d.share)}"></i></div></div>`).join("")+
  (r.achievements.length?`<h3 style="margin-top:10px">Achievements</h3>`+r.achievements.map(a=>`<div class="ach">🏅 ${a.text}</div>`).join(""):"")+
  `<h3 style="margin-top:14px">Highscores</h3>${scoreTable()}<button class="primary" id="again">Play again</button>`);
  $("again").onclick=showStart;
}
function showStart(){
  openModal(`<h2>🏛️ Rise to Mayor</h2><p>The city of Nyhavn needs a new mayor. You have <b>20 weeks</b>, little money and big dreams. Spend your action points wisely, choose your platform, hire help and handle the events – including a TV debate and a late scandal.</p>
  <input type="text" id="nm" maxlength="24" placeholder="Your name" value="${(G&&G.name)||""}">
  <div class="diff"><button data-df="easy">Easy</button><button data-df="normal" class="on">Normal</button><button data-df="hard">Hard</button></div>
  <button class="primary" id="go">Start the campaign</button>
  <h3 style="margin-top:14px">Highscores</h3>${scoreTable()}`);
  pickedDiff="normal";
  document.querySelectorAll("[data-df]").forEach(b=>b.onclick=()=>{pickedDiff=b.dataset.df;document.querySelectorAll("[data-df]").forEach(x=>x.classList.toggle("on",x===b))});
  $("go").onclick=async()=>{const j=await api("/api/new",{name:$("nm").value,difficulty:pickedDiff});if(j){closeModal();toast("The campaign is underway – good luck!");render()}};
}

(async()=>{
  const j=await api("/api/state");
  if(j&&j.game&&!j.game.over){render()} else {if(j&&j.game)render(); showStart()}
})();
</script>
</body>
</html>
"""


def main():
    port = int(sys.argv[1]) if len(sys.argv) > 1 else int(os.environ.get("PORT", 8000))
    server = ThreadingHTTPServer(("0.0.0.0", port), Handler)
    url = f"http://localhost:{port}"
    print(f"Rise to Mayor is running at {url}  (Ctrl+C to stop)")
    if os.environ.get("NO_BROWSER") != "1":
        threading.Timer(0.8, lambda: webbrowser.open(url)).start()
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\nSee you next time!")
        server.server_close()


if __name__ == "__main__":
    main()
 rise_to_mayor (1).py…]()
