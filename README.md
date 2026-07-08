# referral-ranking
Canal Employee Referral Slack 
#!/usr/bin/env python3
"""
Referral Ranking -> Slack (v2: Block Kit + mensaje fijo que se actualiza)
-------------------------------------------------------------------------
Cada ejecucion:
  1. Consulta HubSpot los deals con origen "Referral employee" o "Referral Founder"
     que estan FIRMADOS (etapas: Cotizacion Firmada, Ganado, Facturando).
  2. Cuenta SOLO los que se firmaron por primera vez desde START_DATE.
  3. Agrega por EMPLEADO QUE REFIERE: nº de deals + importe € total.
  4. Publica el Top 3 con Block Kit (header, medallas, premio con imagen).
  5. Si ya existe el mensaje fijado del ranking, LO EDITA (chat.update).
     Si no existe, lo postea y lo fija al canal (pins.add).
     -> El canal siempre tiene UN solo mensaje con el ranking al dia.

Variables de entorno (secrets en GitHub Actions):
  HUBSPOT_TOKEN     -> token Private App HubSpot (scope: crm.objects.deals.read)
  SLACK_BOT_TOKEN   -> token de bot de Slack. Scopes: chat:write, pins:read, pins:write.
                       El bot debe estar en el canal (/invite @bot).
  SLACK_CHANNEL_ID  -> ID del canal (ej. C0BE1UCU360)
"""

import os
import sys
from datetime import date

import requests

# ----------------------------- CONFIG -----------------------------

START_DATE = "2026-06-29"  # YYYY-MM-DD. Solo cuentan firmados desde aqui.

# Deals que SIEMPRE cuentan aunque se firmaran antes de START_DATE.
ALWAYS_INCLUDE_DEAL_IDS = {
    "507940445382",  # Tailos Capital - Propuesta Completa (excepcion solicitada)
}

# ---- PREMIO (edita estas dos lineas cuando cambie el premio) ----
PRIZE_TEXT = "🎁 *Premio del trimestre*\nEl nº 1 del ranking al cierre del trimestre se lleva el premio. ¡A por él!"
PRIZE_IMAGE_URL = ""  # URL PUBLICA de la imagen del premio (https://...). Vacio = sin imagen.

ORIGEN_VALUES = ["Referral employee", "Referral Founder"]
ORIGEN_PROP = "origen_del_deal"
REFERRER_PROPS = ["empleado_que_refiere", "founder_que_refiere"]
SIGNED_STAGES = ["4101805270", "4101805271", "4939795658"]  # Cot. Firmada, Ganado, Facturando
DATE_ENTERED_PROPS = [f"hs_v2_date_entered_{s}" for s in SIGNED_STAGES]
TOP_N = 3

# Marcador para localizar el mensaje fijado del ranking (va en el texto fallback).
MARKER = "🏆 Referral Ranking"

HUBSPOT_TOKEN = os.environ["HUBSPOT_TOKEN"]
SLACK_BOT_TOKEN = os.environ["SLACK_BOT_TOKEN"]
SLACK_CHANNEL_ID = os.environ["SLACK_CHANNEL_ID"]

HS_BASE = "https://api.hubapi.com"
HS_HEADERS = {"Authorization": f"Bearer {HUBSPOT_TOKEN}", "Content-Type": "application/json"}
SLACK_HEADERS = {
    "Authorization": f"Bearer {SLACK_BOT_TOKEN}",
    "Content-Type": "application/json; charset=utf-8",
}

# ----------------------------- HUBSPOT -----------------------------


def fetch_signed_referral_deals():
    url = f"{HS_BASE}/crm/v3/objects/deals/search"
    deals = []
    after = None
    properties = ["dealname", "amount"] + REFERRER_PROPS + DATE_ENTERED_PROPS

    while True:
        payload = {
            "filterGroups": [
                {
                    "filters": [
                        {"propertyName": ORIGEN_PROP, "operator": "IN", "values": ORIGEN_VALUES},
                        {"propertyName": "dealstage", "operator": "IN", "values": SIGNED_STAGES},
                    ]
                }
            ],
            "properties": properties,
            "limit": 100,
        }
        if after:
            payload["after"] = after

        resp = requests.post(url, headers=HS_HEADERS, json=payload, timeout=30)
        resp.raise_for_status()
        data = resp.json()
        deals.extend(data.get("results", []))
        after = data.get("paging", {}).get("next", {}).get("after")
        if not after:
            break

    return deals


def earliest_signed_date(props):
    dates = [props.get(p) for p in DATE_ENTERED_PROPS if props.get(p)]
    return min(dates) if dates else None


# ----------------------------- RANKING -----------------------------


def build_ranking(deals):
    cutoff = START_DATE
    agg = {}
    sin_asignar = 0

    for d in deals:
        props = d.get("properties", {})
        deal_id = str(d.get("id", ""))
        signed = earliest_signed_date(props)
        if deal_id not in ALWAYS_INCLUDE_DEAL_IDS:
            if not signed or signed[:10] < cutoff:
                continue

        referrer = next((props.get(p).strip() for p in REFERRER_PROPS if (props.get(p) or "").strip()), "")
        if not referrer:
            sin_asignar += 1
            continue

        try:
            amount = float(props.get("amount") or 0)
        except (TypeError, ValueError):
            amount = 0.0

        bucket = agg.setdefault(referrer, {"count": 0, "total": 0.0})
        bucket["count"] += 1
        bucket["total"] += amount

    ranking = [{"name": n, "count": v["count"], "total": v["total"]} for n, v in agg.items()]
    ranking.sort(key=lambda x: x["total"], reverse=True)
    return ranking, sin_asignar


# ----------------------------- MENSAJE (Block Kit) -----------------------------


def fmt_eur(n):
    s = f"{n:,.2f}".replace(",", "X").replace(".", ",").replace("X", ".")
    return f"{s} €"


def build_blocks(ranking, sin_asignar):
    medals = ["🥇", "🥈", "🥉"]
    hoy = date.today().strftime("%d/%m/%Y")

    if not ranking:
        body = "_Aún no hay referrals firmados con empleado asignado. ¡A por ellos!_ 🚀"
    else:
        lines = []
        for i, r in enumerate(ranking[:TOP_N]):
            medal = medals[i] if i < len(medals) else f"{i+1}."
            plural = "deal" if r["count"] == 1 else "deals"
            lines.append(f"{medal}  *{r['name']}*  ·  {fmt_eur(r['total'])}  ({r['count']} {plural})")
        body = "\n".join(lines)

    prize_block = {"type": "section", "text": {"type": "mrkdwn", "text": PRIZE_TEXT}}
    if PRIZE_IMAGE_URL:
        prize_block["accessory"] = {
            "type": "image",
            "image_url": PRIZE_IMAGE_URL,
            "alt_text": "Premio",
        }

    blocks = [
        {"type": "header", "text": {"type": "plain_text", "text": "🏆 Referral Ranking", "emoji": True}},
        {
            "type": "context",
            "elements": [
                {
                    "type": "mrkdwn",
                    "text": f"Origen *Referral Employee / Founder* firmados desde el {START_DATE}"
                            f"  ·  🔄 Actualizado el {hoy}",
                }
            ],
        },
        {"type": "divider"},
        {"type": "section", "text": {"type": "mrkdwn", "text": body}},
        {"type": "divider"},
        prize_block,
    ]

    if sin_asignar:
        blocks.append(
            {
                "type": "context",
                "elements": [
                    {
                        "type": "mrkdwn",
                        "text": f"⚠️ {sin_asignar} deal(s) firmados sin 'Empleado que refiere' — no cuentan.",
                    }
                ],
            }
        )

    # Texto fallback (notificaciones y busqueda del mensaje fijado)
    fallback = f"{MARKER} — Top {TOP_N} actualizado el {hoy}"
    return blocks, fallback


# ----------------------------- SLACK -----------------------------


def slack_call(method, http="POST", **kwargs):
    url = f"https://slack.com/api/{method}"
    if http == "GET":
        resp = requests.get(url, headers=SLACK_HEADERS, params=kwargs, timeout=30)
    else:
        resp = requests.post(url, headers=SLACK_HEADERS, json=kwargs, timeout=30)
    data = resp.json()
    return data


def find_pinned_ranking_ts():
    """Busca entre los fijados del canal el mensaje del ranking (por el MARKER)."""
    data = slack_call("pins.list", http="GET", channel=SLACK_CHANNEL_ID)
    if not data.get("ok"):
        print(f"Aviso pins.list: {data.get('error')} (se posteara mensaje nuevo)", file=sys.stderr)
        return None
    for item in data.get("items", []):
        msg = item.get("message") or {}
        if MARKER in (msg.get("text") or ""):
            return msg.get("ts")
    return None


def post_or_update(blocks, fallback):
    ts = find_pinned_ranking_ts()

    if ts:
        data = slack_call("chat.update", channel=SLACK_CHANNEL_ID, ts=ts,
                          text=fallback, blocks=blocks)
        if data.get("ok"):
            print("Mensaje fijado actualizado.")
            return
        print(f"chat.update fallo ({data.get('error')}), posteando nuevo...", file=sys.stderr)

    data = slack_call("chat.postMessage", channel=SLACK_CHANNEL_ID,
                      text=fallback, blocks=blocks, unfurl_links=False)
    if not data.get("ok"):
        print(f"ERROR Slack: {data.get('error')}", file=sys.stderr)
        sys.exit(1)

    new_ts = data.get("ts")
    pin = slack_call("pins.add", channel=SLACK_CHANNEL_ID, timestamp=new_ts)
    if pin.get("ok") or pin.get("error") == "already_pinned":
        print("Mensaje posteado y fijado al canal.")
    else:
        print(f"Mensaje posteado pero no se pudo fijar: {pin.get('error')}", file=sys.stderr)


# ----------------------------- MAIN -----------------------------


def main():
    deals = fetch_signed_referral_deals()
    print(f"Deals firmados con origen Referral (Employee/Founder): {len(deals)}")
    ranking, sin_asignar = build_ranking(deals)
    for r in ranking[:TOP_N]:
        print(f"  {r['name']}: {r['count']} deals, {r['total']:.2f} EUR")
    if sin_asignar:
        print(f"  (sin empleado asignado: {sin_asignar})")
    blocks, fallback = build_blocks(ranking, sin_asignar)
    post_or_update(blocks, fallback)


if __name__ == "__main__":
    main()
