#!/usr/bin/env python3
"""
build_dashboard.py
Fetches live data from Google Sheets and generates index.html.
Triggered automatically by GitHub Actions on a schedule or manual run.
To update the data source: change SHEETS_CSV_URL below.
"""
import json, datetime, sys, os, io
import pandas as pd
import urllib.request

# ── UPDATE THIS URL if your Google Sheet changes ─────────────────────────────
SHEETS_CSV_URL = "https://docs.google.com/spreadsheets/d/e/2PACX-1vRspPg-SIm0rB3hK5GMvM28d_8vT5aB7EsIdHbm4Z8G67ZsER_ettpwPHsNjpL4PKUO-bsujoLgSnz4/pub?output=csv&gid=0"

OUT_FILE = os.path.join(os.path.dirname(__file__), '..', 'index.html')

def load_data():
    for attempt in range(3):
        try:
            print(f"Fetching from Google Sheets (attempt {attempt+1})...")
            req = urllib.request.Request(SHEETS_CSV_URL, headers={"User-Agent":"Mozilla/5.0"})
            with urllib.request.urlopen(req, timeout=30) as resp:
                raw = resp.read().decode('utf-8')
            lines = raw.split('\n')
            hi = 0
            for i, line in enumerate(lines[:10]):
                if 'Part Number' in line and 'ECO' in line:
                    hi = i; break
                if 'Date Added' in line and 'Description' in line:
                    hi = i; break
            df = pd.read_csv(io.StringIO('\n'.join(lines[hi:])))
            df = df.fillna('')
            print(f"Fetched {len(df)} rows (header at row {hi})")
            return df
        except Exception as e:
            print(f"Attempt {attempt+1} failed: {e}")
    print("ERROR: Could not fetch from Google Sheets")
    sys.exit(1)

def get_change_type(row, col):
    if col not in row.index: return 'Rev Change'
    new_pn = str(row[col]).strip().replace('.0','')
    return 'New Part Number' if new_pn and new_pn not in ['', 'nan', 'N/A', 'NaN'] else 'Rev Change'

def get_impl_type(row):
    def sg(col):
        return str(row[col]).strip().lower() if col in row.index else ''
    iv = sg('Implementation at Venture')
    if 'phase' in iv: return 'Phase-In'
    if 'cut'   in iv: return 'Cut-In'
    cs = sg('Change Status')
    if 'phase' in cs: return 'Phase-In'
    notes = sg('Disposition/Implementation Notes')
    if 'phase' in notes: return 'Phase-In'
    if 'cut in' in notes or 'cut-in' in notes: return 'Cut-In'
    return 'Not Specified'

def fmt_date(v):
    if hasattr(v, 'strftime'): return v.strftime('%Y-%m-%d')
    s = str(v).strip()
    return s if s not in ['', 'nan', 'NaT'] else ''

def clean(v):
    s = str(v).strip()
    return '' if s in ['nan', 'NaN'] else s

def clean_pn(v):
    s = str(v).strip()
    if s.endswith('.0'): s = s[:-2]
    return '' if s in ['nan', 'N/A', 'NaN', ''] else s

def sanitize(v):
    if not isinstance(v, str): return v
    return (v.replace('\r\n',' ').replace('\n',' ').replace('\r',' ')
             .replace('\t',' ').replace('</script>','<\\/script>').strip())

def build():
    df = load_data()
    # Normalize column names - strip whitespace
    df.columns = [str(c).strip() for c in df.columns]
    print("Columns:", df.columns.tolist())

    # Find new part number column (name may vary)
    np_col = next((c for c in df.columns if 'New Part Number' in c), None)
    if not np_col:
        print("ERROR: Could not find New Part Number column")
        print("Columns:", df.columns.tolist())
        sys.exit(1)
    print(f"Using new PN column: '{np_col}'")

    records = []
    for _, r in df.iterrows():
        pn  = clean_pn(r['Part Number'])
        np_ = clean_pn(r[np_col])
        if not pn and not np_: continue
        bp = sanitize(clean(r['PL Bridge PO'] if 'PL Bridge PO' in r.index else (r['Bridge PO'] if 'Bridge PO' in r.index else '')))

        # Optional columns (may not exist in older versions)
        def safe(col):
            return clean(r[col]) if col in df.columns else ''

        # Use g() to safely get any column by checking multiple possible names
        def g(primary, *fallbacks):
            for col in [primary] + list(fallbacks):
                if col in r.index and str(r[col]).strip() not in ['','nan','NaN']:
                    return r[col]
            return ''

        rec = {
            'dateAdded':          fmt_date(g('Date Added')),
            'partNumber':         pn,
            'newPartNumber':      np_,
            'description':        sanitize(clean(g('Description'))),
            'rev':                clean(g('REV (After ECO)','Rev (After ECO)','REV')),
            'ecoNumber':          clean(g('ECO #','ECO#','ECO Number')),
            'ecoStatus':          clean(g('ECO Status')),
            'changeStatus':       clean(g('Change Status')),
            'changeType':         get_change_type(r, np_col),
            'implementationType': get_impl_type(r),
            'implAtVenture':      sanitize(clean(g('Implementation at Venture'))),
            'rdOwner':            clean(g('R&D Owner','R&D Owner ')),
            'bridgePO':           bp,
            'bridgeQty':          clean_pn(g('PL Bridge QTY','Bridge QTY')),
            'hasBridgePO':        'Yes' if bp else 'No',
            'remarks':            sanitize(clean(g('Remarks'))),
            'singaporeOwner':     clean(g('Singapore Owner','SG Owner')),
            'implNotes':          sanitize(clean(g('Disposition/Implementation Notes','Implementation Notes'))),
            'ecoReleaseDate':     fmt_date(g('ECO Release Date','ECO Release')),
            'implDate':           fmt_date(g('Implementation Date (for Phase-In parts at Venture)','Implementation Date')),
            'nirStatus':          clean(g('NIR Status (PL)','NIR Status')),
            'buyerName':          clean(g('Buyer Name ','Buyer Name','Buyer')),
            'openPO':             clean(g('Open PO')),
            'plAction':           sanitize(clean(g('PL - Procurement Action','Procurement Action'))),
        }
        records.append(rec)

    print(f"Parsed {len(records)} records")

    edata = json.dumps(records, separators=(',',':'), ensure_ascii=True)
    assert '\n' not in edata, "Newlines in JSON!"
    assert '</script>' not in edata.lower(), "Script tag in JSON!"

    snap_date = datetime.datetime.now().strftime('%d %b %Y %H:%M UTC')
    html = generate_html(edata, snap_date, len(records))

    with open(OUT_FILE, 'w') as f:
        f.write(html)
    print(f"Written {OUT_FILE} ({len(html)//1024}KB)")

def generate_html(edata, snap_date, total):
    return """<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Aurora Pilot Build CCB Dashboard</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{background:#0f172a;font-family:'Courier New',monospace;color:#e2e8f0;min-height:100vh}
button,select,input{font-family:inherit;cursor:pointer}
::-webkit-scrollbar{width:6px;height:6px}::-webkit-scrollbar-track{background:#1e293b}::-webkit-scrollbar-thumb{background:#334155;border-radius:3px}
#hdr{background:linear-gradient(135deg,#0f172a,#1e293b);border-bottom:1px solid #1e40af;padding:16px 24px}
#hdr-top{display:flex;align-items:center;justify-content:space-between;flex-wrap:wrap;gap:8px}
.eco-lbl{font-size:11px;color:#64748b;letter-spacing:3px;text-transform:uppercase;display:flex;align-items:center;gap:8px;margin-bottom:4px}
#hdr h1{font-size:20px;font-weight:700;color:#f8fafc}
.sub{font-size:11px;color:#475569;margin-top:3px}
#live-bar{display:flex;align-items:center;gap:10px;flex-wrap:wrap}
.ldot{width:8px;height:8px;background:#4ade80;border-radius:50%;box-shadow:0 0 6px #4ade80;animation:pulse 2s infinite}
.llbl{font-size:10px;color:#4ade80;letter-spacing:1px}
#lupd{font-size:10px;color:#4ade80}
#rbtn{padding:5px 12px;background:#1e293b;border:1px solid #334155;border-radius:4px;color:#60a5fa;font-size:11px;font-weight:600}
#bnr{background:linear-gradient(90deg,#1e3a5f,#1e293b);border-bottom:1px solid #1e40af;padding:10px 24px;display:flex;align-items:center;gap:16px;flex-wrap:wrap}
#gtnum{font-size:34px;font-weight:800;color:#f8fafc;line-height:1}
#gtlbl{font-size:12px;color:#93c5fd;font-weight:700;text-transform:uppercase;letter-spacing:2px;margin-left:8px}
.bdiv{width:1px;height:30px;background:#334155;flex-shrink:0}
#bstats{display:flex;gap:16px;flex-wrap:wrap}
.bs{display:flex;align-items:center;gap:5px}
.bs .bv{font-size:17px;font-weight:700}
.bs .bn{font-size:9px;color:#94a3b8;text-transform:uppercase}
.bs .bp{font-size:9px;color:#475569}
#vblk{margin-left:auto;text-align:right}
#vblk .vl{font-size:9px;color:#475569;text-transform:uppercase;letter-spacing:1px}
#vcnt{font-size:18px;font-weight:700;color:#fbbf24}
#chart-sec{background:#0f172a;border-bottom:1px solid #1e293b;padding:16px 24px}
#chart-hdr{display:flex;align-items:center;justify-content:space-between;margin-bottom:12px;flex-wrap:wrap;gap:8px}
.ch-title{font-size:11px;color:#64748b;text-transform:uppercase;letter-spacing:2px;margin-bottom:2px}
.ch-main{font-size:13px;font-weight:700;color:#f8fafc}
.legend{display:flex;gap:14px;font-size:10px}
.leg-item{display:flex;align-items:center;gap:4px;color:#94a3b8}
.leg-dot{width:11px;height:11px;border-radius:2px}
#chart-wrap{overflow-x:auto;padding-bottom:4px}
#srow{display:flex;background:#1e293b;border-bottom:1px solid #1e40af;overflow-x:auto}
.stile{flex:1 1 88px;padding:9px 12px;background:#0f172a;min-width:88px;border-right:1px solid #1e293b}
.stile .sn{font-size:22px;font-weight:800;line-height:1}
.stile .sl{font-size:9px;color:#64748b;margin-top:3px;text-transform:uppercase;letter-spacing:.7px;white-space:nowrap}
#tabs{display:flex;background:#1e293b;border-bottom:1px solid #334155;overflow-x:auto}
.tab{padding:9px 16px;font-size:11px;font-weight:600;border:none;border-bottom:2px solid transparent;background:transparent;color:#64748b;white-space:nowrap}
.tab.active{border-bottom-color:#3b82f6;background:#0f172a;color:#60a5fa}
#flt{display:flex;gap:8px;padding:10px 14px;background:#0f172a;border-bottom:1px solid #1e293b;flex-wrap:wrap;align-items:center}
#srch{flex:1 1 240px;padding:6px 10px;background:#1e293b;border:1px solid #334155;border-radius:4px;color:#e2e8f0;font-size:12px;outline:none}
#srch::placeholder{color:#475569}
.fsel{padding:6px 9px;background:#1e293b;border:1px solid #334155;border-radius:4px;color:#e2e8f0;font-size:11px;outline:none}
#fcnt{font-size:11px;color:#475569;margin-left:auto}
#twrap{overflow:auto;max-height:calc(100vh - 380px)}
table{width:100%;border-collapse:collapse;font-size:11px}
thead tr{background:#1e293b;position:sticky;top:0;z-index:10}
th{padding:7px 9px;text-align:left;color:#94a3b8;font-weight:700;font-size:9px;border-bottom:1px solid #334155;white-space:nowrap;background:#1e293b;text-transform:uppercase;letter-spacing:.4px}
th .ck{color:#3b82f6;margin-right:3px}
tbody tr{cursor:pointer}
tbody tr:nth-child(even){background:#0b1120}
tbody tr:nth-child(odd){background:#0f172a}
tbody tr:hover{background:#1e293b !important}
tbody tr.xrow{background:#1e3a5f !important}
tbody tr.prow{border-left:3px solid #7c3aed}
td{padding:6px 9px;border-bottom:1px solid #1e293b;vertical-align:middle}
.tr{overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.badge{padding:2px 7px;border-radius:99px;font-size:10px;font-weight:600;white-space:nowrap;display:inline-block}
.ct-n{background:#4a044e;color:#f0abfc}.ct-r{background:#0c2a4a;color:#93c5fd}
.im-c{background:#dbeafe;color:#1e40af}.im-p{background:#ede9fe;color:#5b21b6}.im-n{background:#1e293b;color:#64748b;border:1px solid #334155}
.eo-ok{background:#dcfce7;color:#166534}.eo-pn{background:#fef3c7;color:#92400e}.eo-op{background:#fee2e2;color:#991b1b}.eo-na{background:#1e293b;color:#475569}
.cs-dn{background:#dcfce7;color:#166534}.cs-op{background:#fef9c3;color:#854d0e}.cs-pi{background:#ede9fe;color:#5b21b6}.cs-tb{background:#fee2e2;color:#991b1b}.cs-na{background:#1e293b;color:#475569;border:1px solid #334155}
.xcell{background:#1e3a5f;padding:14px 20px;border-bottom:2px solid #3b82f6}
.xgrid{display:grid;grid-template-columns:repeat(auto-fill,minmax(210px,1fr));gap:12px}
.xk{font-size:9px;color:#475569;text-transform:uppercase;letter-spacing:1px;margin-bottom:2px}
.xv{color:#e2e8f0;font-size:12px;font-weight:600}
.xfull{grid-column:1/-1}
.nbox{color:#cbd5e1;font-size:12px;line-height:1.5;background:#0f172a;padding:8px 12px;border-radius:4px;margin-top:4px}
.nb{border-left:3px solid #3b82f6}.np{border-left:3px solid #f472b6}
#ftr{padding:9px 14px;background:#0b1120;border-top:1px solid #1e293b;display:flex;justify-content:space-between;align-items:center;font-size:10px;color:#334155;flex-wrap:wrap;gap:6px}
#ebtns{display:flex;gap:6px;align-items:center}
.eb1{padding:4px 12px;background:#1e293b;border:1px solid #334155;border-radius:4px;color:#34d399;font-size:11px;font-weight:700}
.eb2{padding:4px 12px;background:#1e3a5f;border:1px solid #1e40af;border-radius:4px;color:#60a5fa;font-size:11px;font-weight:700}
.eb3{padding:4px 12px;background:#2d1f4a;border:1px solid #5b21b6;border-radius:4px;color:#a78bfa;font-size:11px;font-weight:700}
#wk-tip{position:fixed;background:#1e293b;border:1px solid #334155;border-radius:6px;padding:8px 12px;font-size:11px;color:#e2e8f0;pointer-events:none;z-index:999;font-family:'Courier New',monospace;line-height:1.7;display:none}
#load{display:flex;flex-direction:column;align-items:center;justify-content:center;height:100vh;gap:14px;color:#60a5fa}
.spin{font-size:32px;animation:spin 1s linear infinite}
@keyframes spin{from{transform:rotate(0)}to{transform:rotate(360deg)}}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.4}}
</style>
</head>
<body>
<div id="wk-tip"></div>
<div id="load"><div class="spin">&#8635;</div><div style="font-size:13px;letter-spacing:2px">LOADING DASHBOARD&hellip;</div></div>
<div id="app" style="display:none">
<div id="hdr"><div id="hdr-top">
  <div>
    <div class="eco-lbl"><div style="width:9px;height:9px;background:#3b82f6;border-radius:2px;box-shadow:0 0 7px #3b82f6"></div>Aurora Instrument &middot; Production CCB</div>
    <h1>Pilot Build Implementation Dashboard</h1>
    <div class="sub">Auto-updated on every spreadsheet upload to GitHub</div>
  </div>
  <div id="live-bar">
    <div class="ldot"></div><span class="llbl">LIVE</span>
    <span id="lupd">Updated: """ + snap_date + """</span>
    <button id="rbtn">&#8635; Refresh</button>
  </div>
</div></div>
<div id="bnr">
  <div><span id="gtnum">0</span><span id="gtlbl">Grand Total Parts</span></div>
  <div class="bdiv"></div>
  <div id="bstats"></div>
  <div id="vblk"><div class="vl">Viewing</div><div id="vcnt">0</div></div>
</div>
<div id="chart-sec">
  <div id="chart-hdr">
    <div><div class="ch-title">Weekly Trend</div><div class="ch-main">Total Parts Added by Week</div></div>
    <div class="legend">
      <div class="leg-item"><div class="leg-dot" style="background:#3b82f6"></div>Cut-In</div>
      <div class="leg-item"><div class="leg-dot" style="background:#a78bfa"></div>Phase-In</div>
      <div class="leg-item"><div class="leg-dot" style="background:#334155"></div>Not Specified</div>
    </div>
  </div>
  <div id="chart-wrap"><canvas id="weekChart"></canvas></div>
</div>
<div id="srow"></div>
<div id="tabs"></div>
<div id="flt">
  <input id="srch" type="text" placeholder="Search part number, description, ECO, engineer..."/>
  <select id="f-eco" class="fsel"></select>
  <select id="f-owner" class="fsel"></select>
  <select id="f-status" class="fsel"></select>
  <select id="f-ecost" class="fsel"></select>
  <span id="fcnt"></span>
</div>
<div id="twrap"><table><thead id="thead"></thead><tbody id="tbody"></tbody></table></div>
<div id="ftr">
  <span>Aurora CCB &middot; Pilot Build Dashboard &middot; Click row to expand</span>
  <div id="ebtns">
    <span style="color:#475569;font-size:10px">Export <strong id="ecnt" style="color:#60a5fa">0</strong>:</span>
    <button class="eb1" id="b-csv">&#8595; CSV</button>
    <button class="eb2" id="b-xlsx">&#8595; Excel</button>
    <button class="eb3" id="b-all">&#8595; All (xlsx)</button>
  </div>
</div>
</div>
<script>
try {
var SNAP=""" + '"' + snap_date + '"' + """;
var TABS=["All Parts","Rev Changes","New Part Numbers","Bridge PO","Cut-In","Phase-In","Not Specified"];
var COLS=[
  {k:"A",h:"Date Added",w:100},{k:"B",h:"Old Part #",w:130},{k:"C",h:"New Part #",w:130},
  {k:"D",h:"Description",w:220},{k:"E",h:"Change Type",w:130},{k:"F",h:"Rev",w:60},
  {k:"G",h:"ECO #",w:120},{k:"H",h:"ECO Status",w:120},{k:"I",h:"ECO Release",w:100},
  {k:"J",h:"Implementation",w:115},{k:"K",h:"Change Status",w:155},{k:"L",h:"R&D Engineer",w:140},
  {k:"M",h:"Bridge PO",w:120},{k:"N",h:"Bridge QTY",w:85},{k:"O",h:"NIR Status",w:120},
  {k:"P",h:"SG Owner",w:95},{k:"Q",h:"Impl Date",w:100},{k:"R",h:"Buyer",w:120},
  {k:"S",h:"Open PO",w:100},{k:"T",h:"Remarks",w:240},{k:"U",h:"Impl Notes",w:240}
];
var EMB=""" + edata + """;
var ALL=[],FILT=[],TAB="All Parts",EXP=null;
function esc(s){return String(s||"").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");}
function ecoC(s){var v=(s||"").toLowerCase();if(v==="effective"||v==="complete"||v==="approved")return"eo-ok";if(v.indexOf("approv")>-1)return"eo-pn";if(v.indexOf("open")>-1)return"eo-op";return"eo-na";}
function csC(s){if(!s)return"cs-na";if(s==="No further action required")return"cs-dn";if(s==="Open")return"cs-op";if(s==="Phase-In")return"cs-pi";if(s==="TBC")return"cs-tb";return"cs-na";}
function imC(s){if(s==="Cut-In")return"im-c";if(s==="Phase-In")return"im-p";return"im-n";}
function calcS(d){return{tot:d.length,rev:d.filter(function(r){return r.changeType==="Rev Change";}).length,np:d.filter(function(r){return r.changeType==="New Part Number";}).length,ci:d.filter(function(r){return r.implementationType==="Cut-In";}).length,pi:d.filter(function(r){return r.implementationType==="Phase-In";}).length,ns:d.filter(function(r){return r.implementationType==="Not Specified";}).length,bp:d.filter(function(r){return r.hasBridgePO==="Yes";}).length,op:d.filter(function(r){return r.changeStatus==="Open";}).length,dn:d.filter(function(r){return r.changeStatus==="No further action required";}).length};}
function fillSel(id,opts,lbl){var el=document.getElementById(id),cur=el.value||"All";el.innerHTML=opts.map(function(o){return'<option value="'+esc(o)+'">'+lbl+': '+(o==="All"?"All":esc(o))+'</option>';}).join("");if(opts.indexOf(cur)>-1)el.value=cur;}
function applyF(){
  var q=document.getElementById("srch").value.toLowerCase().trim();
  var eco=document.getElementById("f-eco").value,ecost=document.getElementById("f-ecost").value;
  var own=document.getElementById("f-owner").value,sta=document.getElementById("f-status").value;
  FILT=ALL.filter(function(r){
    if(TAB==="Rev Changes"&&r.changeType!=="Rev Change")return false;
    if(TAB==="New Part Numbers"&&r.changeType!=="New Part Number")return false;
    if(TAB==="Bridge PO"&&r.hasBridgePO!=="Yes")return false;
    if(TAB==="Cut-In"&&r.implementationType!=="Cut-In")return false;
    if(TAB==="Phase-In"&&r.implementationType!=="Phase-In")return false;
    if(TAB==="Not Specified"&&r.implementationType!=="Not Specified")return false;
    if(eco!=="All"&&r.ecoNumber!==eco)return false;
    if(ecost!=="All"){if(ecost==="--Blank"&&r.ecoStatus!=="")return false;if(ecost!=="--Blank"&&r.ecoStatus!==ecost)return false;}
    if(own!=="All"&&r.rdOwner!==own)return false;
    if(sta!=="All"){if(sta==="--Blank"&&r.changeStatus!=="")return false;if(sta!=="--Blank"&&r.changeStatus!==sta)return false;}
    if(q){var h=[r.partNumber,r.newPartNumber,r.description,r.ecoNumber,r.rdOwner,r.bridgePO,r.nirStatus,r.buyerName].join(" ").toLowerCase();if(h.indexOf(q)<0)return false;}
    return true;
  });
  renderT();
  document.getElementById("vcnt").textContent=FILT.length;
  document.getElementById("fcnt").innerHTML='Showing <strong style="color:#fbbf24">'+FILT.length+'</strong> of <strong style="color:#60a5fa">'+ALL.length+'</strong> parts';
  document.getElementById("ecnt").textContent=FILT.length;
}
function render(){
  var s=calcS(ALL);
  document.getElementById("gtnum").textContent=s.tot;
  document.getElementById("bstats").innerHTML=[{l:"Rev Changes",v:s.rev,c:"#34d399"},{l:"New Part #s",v:s.np,c:"#f472b6"},{l:"Cut-In",v:s.ci,c:"#60a5fa"},{l:"Phase-In",v:s.pi,c:"#a78bfa"},{l:"Not Specified",v:s.ns,c:"#64748b"}].map(function(x){return'<div class="bs"><span class="bv" style="color:'+x.c+'">'+x.v+'</span><div><div class="bn">'+x.l+'</div><div class="bp">'+Math.round(x.v/s.tot*100)+'%</div></div></div>';}).join("");
  document.getElementById("srow").innerHTML=[{l:"Total",v:s.tot,c:"#60a5fa"},{l:"Rev Changes",v:s.rev,c:"#34d399"},{l:"New Part #s",v:s.np,c:"#f472b6"},{l:"Cut-In",v:s.ci,c:"#60a5fa"},{l:"Phase-In",v:s.pi,c:"#a78bfa"},{l:"Not Specified",v:s.ns,c:"#64748b"},{l:"Bridge PO",v:s.bp,c:"#fb923c"},{l:"Open",v:s.op,c:"#facc15"},{l:"Complete",v:s.dn,c:"#4ade80"}].map(function(t){return'<div class="stile"><div class="sn" style="color:'+t.c+'">'+t.v+'</div><div class="sl">'+t.l+'</div></div>';}).join("");
  document.getElementById("tabs").innerHTML=TABS.map(function(t){return'<button class="tab'+(t===TAB?" active":"")+'" data-tab="'+t+'">'+t+'</button>';}).join("");
  document.querySelectorAll(".tab").forEach(function(b){b.addEventListener("click",function(){TAB=this.dataset.tab;EXP=null;document.querySelectorAll(".tab").forEach(function(x){x.classList.remove("active");});this.classList.add("active");applyF();});});
  fillSel("f-eco",["All"].concat(Array.from(new Set(ALL.map(function(r){return r.ecoNumber;}).filter(Boolean))).sort()),"ECO");
  fillSel("f-ecost",["All"].concat(Array.from(new Set(ALL.map(function(r){return r.ecoStatus;}).filter(Boolean))).sort()).concat(["--Blank"]),"H - ECO Status");
  fillSel("f-owner",["All"].concat(Array.from(new Set(ALL.map(function(r){return r.rdOwner;}).filter(Boolean))).sort()),"R&D Owner");
  fillSel("f-status",["All"].concat(Array.from(new Set(ALL.map(function(r){return r.changeStatus;}).filter(Boolean))).sort()).concat(["--Blank"]),"K - Change Status");
  document.getElementById("thead").innerHTML="<tr>"+COLS.map(function(c){return'<th style="min-width:'+c.w+'px"><span class="ck">'+c.k+'</span>'+c.h+'</th>';}).join("")+"</tr>";
  applyF();
}
function renderT(){
  var tb=document.getElementById("tbody");
  if(!FILT.length){tb.innerHTML='<tr><td colspan="21" style="padding:36px;text-align:center;color:#475569">No records match filters.</td></tr>';return;}
  var rows=[];
  FILT.forEach(function(r,i){
    var xc=(EXP===i);
    rows.push('<tr class="dr'+(r.implementationType==="Phase-In"?" prow":"")+(xc?" xrow":"")+'" data-i="'+i+'">'
      +'<td style="color:#64748b;white-space:nowrap">'+esc(r.dateAdded||"--")+'</td>'
      +'<td><span style="color:#60a5fa;font-weight:700">'+esc(r.partNumber||"--")+'</span></td>'
      +'<td>'+(r.newPartNumber?'<span style="color:#f472b6;font-weight:700">'+esc(r.newPartNumber)+'</span>':'<span style="color:#334155">--</span>')+'</td>'
      +'<td style="color:#cbd5e1;max-width:220px"><div class="tr">'+esc(r.description)+'</div></td>'
      +'<td><span class="badge '+(r.changeType==="New Part Number"?"ct-n":"ct-r")+'">'+esc(r.changeType)+'</span></td>'
      +'<td style="text-align:center;color:#fbbf24;font-weight:700">'+esc(r.rev||"--")+'</td>'
      +'<td style="color:#a5b4fc;white-space:nowrap">'+esc(r.ecoNumber)+'</td>'
      +'<td><span class="badge '+ecoC(r.ecoStatus)+'">'+esc(r.ecoStatus||"--")+'</span></td>'
      +'<td style="color:#64748b;white-space:nowrap">'+esc(r.ecoReleaseDate||"--")+'</td>'
      +'<td><span class="badge '+imC(r.implementationType)+'">'+esc(r.implementationType)+'</span></td>'
      +'<td><span class="badge '+csC(r.changeStatus)+'">'+esc(r.changeStatus||"--")+'</span></td>'
      +'<td style="color:#94a3b8;white-space:nowrap">'+esc(r.rdOwner||"--")+'</td>'
      +'<td>'+(r.bridgePO?'<span style="color:#fb923c;font-weight:700">'+esc(r.bridgePO)+'</span>':'<span style="color:#334155">--</span>')+'</td>'
      +'<td style="color:#fbbf24;text-align:center">'+esc(r.bridgeQty||"--")+'</td>'
      +'<td style="color:#94a3b8;white-space:nowrap">'+esc(r.nirStatus||"--")+'</td>'
      +'<td style="color:#94a3b8;white-space:nowrap">'+esc(r.singaporeOwner||"--")+'</td>'
      +'<td style="color:#64748b;white-space:nowrap">'+esc(r.implDate||"--")+'</td>'
      +'<td style="color:#94a3b8;white-space:nowrap">'+esc(r.buyerName||"--")+'</td>'
      +'<td style="color:#64748b;white-space:nowrap">'+esc(r.openPO||"--")+'</td>'
      +'<td style="color:#94a3b8;max-width:240px"><div class="tr">'+esc(r.remarks||"--")+'</div></td>'
      +'<td style="color:#64748b;max-width:240px"><div class="tr">'+esc(r.implNotes||"--")+'</div></td>'
      +'</tr>');
    if(xc){
      var flds=[["Old Part #",r.partNumber],["New Part #",r.newPartNumber],["Description",r.description],["Change Type",r.changeType],["Rev",r.rev],["ECO #",r.ecoNumber],["ECO Status",r.ecoStatus],["ECO Release",r.ecoReleaseDate],["Implementation",r.implementationType],["Impl at Venture",r.implAtVenture],["Change Status",r.changeStatus],["R&D Engineer",r.rdOwner],["NIR Status",r.nirStatus],["SG Owner",r.singaporeOwner],["Bridge PO",r.bridgePO],["Bridge QTY",r.bridgeQty],["Date Added",r.dateAdded],["Impl Date",r.implDate],["Buyer",r.buyerName],["Open PO",r.openPO],["PL Action",r.plAction]];
      rows.push('<tr><td colspan="21" class="xcell"><div class="xgrid">'+flds.map(function(f){return'<div><div class="xk">'+f[0]+'</div><div class="xv">'+esc(f[1]||"--")+'</div></div>';}).join("")+'<div class="xfull"><div class="xk">Implementation Notes</div><div class="nbox nb">'+esc(r.implNotes||"--")+'</div></div>'+(r.remarks?'<div class="xfull"><div class="xk">Remarks</div><div class="nbox np">'+esc(r.remarks)+'</div></div>':"")+(r.plAction?'<div class="xfull"><div class="xk">PL Procurement Action</div><div class="nbox" style="border-left:3px solid #fb923c">'+esc(r.plAction)+'</div></div>':"")+'</div></td></tr>');
    }
  });
  tb.innerHTML=rows.join("");
  document.querySelectorAll(".dr").forEach(function(row){row.addEventListener("click",function(){var i=parseInt(this.dataset.i);EXP=(EXP===i)?null:i;renderT();});});
}
function buildChart(){
  var canvas=document.getElementById("weekChart");
  if(!canvas)return;
  var wrap=document.getElementById("chart-wrap");
  var ctx=canvas.getContext("2d");
  var weekMap={};
  ALL.forEach(function(r){
    if(!r.dateAdded)return;
    var p=r.dateAdded.split("-");if(p.length<3)return;
    var d=new Date(parseInt(p[0]),parseInt(p[1])-1,parseInt(p[2]));
    var day=d.getDay(),diff=d.getDate()-day+(day===0?-6:1);d.setDate(diff);
    var wk=d.getFullYear()+"-"+String(d.getMonth()+1).padStart(2,"0")+"-"+String(d.getDate()).padStart(2,"0");
    if(!weekMap[wk])weekMap[wk]={ci:0,pi:0,ns:0};
    if(r.implementationType==="Cut-In")weekMap[wk].ci++;
    else if(r.implementationType==="Phase-In")weekMap[wk].pi++;
    else weekMap[wk].ns++;
  });
  var weeks=Object.keys(weekMap).sort();
  var maxVal=0;
  weeks.forEach(function(w){var t=weekMap[w].ci+weekMap[w].pi+weekMap[w].ns;if(t>maxVal)maxVal=t;});
  maxVal=Math.ceil(maxVal/10)*10||10;
  var BW=56,GAP=18,PL=40,PR=16,PT=20,PB=46;
  var TW=Math.max((wrap.clientWidth||640),weeks.length*(BW+GAP)+PL+PR+GAP);
  var H=200;
  canvas.width=TW;canvas.height=H;canvas.style.width=TW+"px";canvas.style.height=H+"px";
  ctx.clearRect(0,0,TW,H);
  var CH=H-PT-PB;
  for(var g=0;g<=4;g++){
    var yg=PT+CH-(g/4)*CH;
    ctx.strokeStyle="#1e293b";ctx.lineWidth=1;
    ctx.beginPath();ctx.moveTo(PL,yg);ctx.lineTo(TW-PR,yg);ctx.stroke();
    ctx.fillStyle="#475569";ctx.font="9px 'Courier New',monospace";ctx.textAlign="right";
    ctx.fillText(Math.round(g/4*maxVal),PL-5,yg+3);
  }
  var tip=document.getElementById("wk-tip");
  weeks.forEach(function(wk,i){
    var x=PL+GAP/2+i*(BW+GAP);
    var d=weekMap[wk],total=d.ci+d.pi+d.ns;
    var segs=[{v:d.ci,c:"#3b82f6"},{v:d.pi,c:"#a78bfa"},{v:d.ns,c:"#334155"}];
    var yOff=0;
    segs.forEach(function(seg,si){
      if(!seg.v)return;
      var bh=Math.max(2,(seg.v/maxVal)*CH);
      var by=PT+CH-yOff-bh;
      var isTop=segs.slice(si+1).every(function(s){return!s.v;});
      ctx.fillStyle=seg.c;ctx.beginPath();
      if(isTop&&bh>6){var r=3;ctx.moveTo(x+r,by);ctx.lineTo(x+BW-r,by);ctx.quadraticCurveTo(x+BW,by,x+BW,by+r);ctx.lineTo(x+BW,by+bh);ctx.lineTo(x,by+bh);ctx.lineTo(x,by+r);ctx.quadraticCurveTo(x,by,x+r,by);}
      else{ctx.rect(x,by,BW,bh);}
      ctx.fill();
      if(bh>16){ctx.fillStyle="#fff";ctx.font="bold 10px 'Courier New',monospace";ctx.textAlign="center";ctx.fillText(seg.v,x+BW/2,by+bh/2+4);}
      yOff+=bh;
    });
    if(total>0){ctx.fillStyle="#f8fafc";ctx.font="bold 11px 'Courier New',monospace";ctx.textAlign="center";ctx.fillText(total,x+BW/2,PT+CH-yOff-5);}
    ctx.fillStyle="#64748b";ctx.font="9px 'Courier New',monospace";ctx.textAlign="center";
    ctx.fillText("w/c "+wk.slice(5),x+BW/2,H-PB+14);
    ctx.fillText(wk.slice(0,4),x+BW/2,H-PB+25);
  });
  canvas.onmousemove=function(e){
    var rect=canvas.getBoundingClientRect(),mx=(e.clientX-rect.left)*(canvas.width/rect.width),found=false;
    weeks.forEach(function(wk,i){
      var x=PL+GAP/2+i*(BW+GAP);
      if(mx>=x&&mx<=x+BW){found=true;var d=weekMap[wk];
        tip.innerHTML="<strong style='color:#60a5fa'>Week of "+wk+"</strong><br><span style='color:#3b82f6'>&#9632;</span> Cut-In: <strong>"+d.ci+"</strong><br><span style='color:#a78bfa'>&#9632;</span> Phase-In: <strong>"+d.pi+"</strong><br><span style='color:#64748b'>&#9632;</span> Not Specified: <strong>"+d.ns+"</strong><br>Total: <strong style='color:#fbbf24'>"+(d.ci+d.pi+d.ns)+"</strong>";
        tip.style.display="block";tip.style.left=(e.clientX+14)+"px";tip.style.top=(e.clientY-10)+"px";}
    });
    if(!found)tip.style.display="none";
  };
  canvas.onmouseleave=function(){tip.style.display="none";};
}
function expData(data,fmt){
  var hdr=["Date Added","Old Part #","New Part #","Description","Change Type","Rev","ECO #","ECO Status","ECO Release","Implementation","Change Status","R&D Engineer","Bridge PO","Bridge QTY","NIR Status","SG Owner","Impl Date","Buyer","Open PO","Remarks","Impl Notes","PL Action"];
  var rows=data.map(function(r){return[r.dateAdded,r.partNumber,r.newPartNumber,r.description,r.changeType,r.rev,r.ecoNumber,r.ecoStatus,r.ecoReleaseDate,r.implementationType,r.changeStatus,r.rdOwner,r.bridgePO,r.bridgeQty,r.nirStatus,r.singaporeOwner,r.implDate,r.buyerName,r.openPO,r.remarks,r.implNotes,r.plAction];});
  if(fmt==="csv"){var csv=[hdr].concat(rows).map(function(row){return row.map(function(v){return'"'+String(v||"").replace(/"/g,'""')+'"';}).join(",");}).join("\\n");var a=document.createElement("a");a.href=URL.createObjectURL(new Blob([csv],{type:"text/csv"}));a.download="Aurora_CCB.csv";a.click();}
  else{var ws=XLSX.utils.aoa_to_sheet([hdr].concat(rows));ws["!cols"]=[10,14,14,28,14,6,13,12,11,13,20,16,13,9,13,10,11,14,11,32,36,36].map(function(w){return{wch:w};});var wb=XLSX.utils.book_new();XLSX.utils.book_append_sheet(wb,ws,"CCB");XLSX.writeFile(wb,"Aurora_CCB.xlsx");}
}
document.addEventListener("DOMContentLoaded",function(){
  ALL=EMB.slice();
  document.getElementById("load").style.display="none";
  document.getElementById("app").style.display="block";
  render();buildChart();
  document.getElementById("rbtn").addEventListener("click",function(){location.reload();});
  document.getElementById("srch").addEventListener("input",function(){EXP=null;applyF();});
  ["f-eco","f-ecost","f-owner","f-status"].forEach(function(id){document.getElementById(id).addEventListener("change",function(){EXP=null;applyF();});});
  document.getElementById("b-csv").addEventListener("click",function(){expData(FILT,"csv");});
  document.getElementById("b-xlsx").addEventListener("click",function(){expData(FILT,"xlsx");});
  document.getElementById("b-all").addEventListener("click",function(){expData(ALL,"xlsx");});
  window.addEventListener("resize",buildChart);
});
} catch(e){
  document.getElementById("load").style.display="none";
  document.body.innerHTML='<div style="padding:40px;color:#f87171;font-size:14px;font-family:monospace">Dashboard error: '+e.message+'<br><br>'+e.stack+'</div>';
  console.error(e);
}
</script>
</body></html>"""

if __name__ == "__main__":
    build()
