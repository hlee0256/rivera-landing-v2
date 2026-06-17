# Handover — Rivera project cards

## Read me first
- **Correct repo: `hlee0256/rivera-website`.** A previous session was opened on the WRONG repo (`rivera-landing-v2`) and is walled off from `rivera-website`, so the work must be (re)applied in a session opened on `rivera-website`.
- Claude on the web runs in a **cloud container** — it can only see files **committed to the repo**, never the user's Mac (`/Users/hoanganh/Desktop/...`).
- **Project photos live on the user's Desktop at `Rivera-Website/Projects/<project>/...`.** They must be **committed + pushed into `rivera-website`** before the agent can use them. A local-only folder is invisible to the agent.

## Goal
Rebuild the **project cards** on the Rivera site to match a reference design (a real-estate listing card). Each card:
1. **Cover photo** with two overlaid pills:
   - **Top-left = standardized project status pill** (this was the user's main ask).
   - **Top-right = sales-status pill** (e.g. "SELLING NOW" with a coloured dot).
2. **Title** (project name), **address**, **2-line truncated description**.
3. A **pricing block**: one row per configuration showing bed / bath / car icons + price.

## Decisions already locked in
- **Standardized status taxonomy (3 values only):** `under-construction`, `pre-construction`, `completed`. English labels on the pill.
- **Status pill colours = on-brand** (NOT the reference's indigo): under-construction = brass, pre-construction = forest green, completed = green.
- **Sales pill:** `selling` ("Selling now", green dot) / `coming-soon` (brass dot) / `sold-out` (red dot).
- **Pricing rule (important):** the user supplies a **raw price list**; the AI extracts rows and shows the **LOWEST price per phân khúc (segment)**. A segment = unique `beds-baths-cars` combo. Collapse duplicates and show ONE row per segment as **"from $X"**. (In the reference, a 1bd/1ba appeared twice at $865k and $850k → must collapse to a single `from $850,000` row.)
- **Workflow:** user pastes/produces a price list per project → AI fills that project's `listings` array → renderer does the grouping automatically.

## The 5 real projects (verified metadata — addresses & status confirmed)
| Project | Address | Status |
|---|---|---|
| Aluna at Collins Wharf | 989 Collins Street, Docklands VIC 3008 | pre-construction |
| Ancora at Collins Wharf | 971 Collins Street, Docklands VIC 3008 | under-construction |
| 380 Melbourne | 371 Little Lonsdale Street, Melbourne VIC 3000 | **completed** (user-confirmed) |
| Aspire Melbourne | 299 King Street, Melbourne VIC 3000 | completed |
| 671 Chapel Street, South Yarra | 671 Chapel Street, South Yarra VIC 3141 | under-construction |

All currently `sale: 'selling'`. (Sources: Lendlease/Collins Wharf, ICD Property, CASA/Kapitol, project sites.)

## Open items for the new session
1. **Commit the `Projects/` photos into `rivera-website`** (or have the agent do the git add/commit/push once they're in the working tree).
2. Agent **browses each project folder, picks the best hero shot**, sets each card's `image` to that path.
3. **Pricing is still empty** — user must provide a price list per project; agent fills `listings` (rows of `{beds,baths,cars,price}`, duplicates fine). Until then cards show a "Bảng giá: liên hệ Rivera" placeholder.
4. NOTE: `rivera-website` may have different HTML/structure than the code below — adapt the markup/section, but keep the format, taxonomy, colours, and pricing rule identical.

---

## Working code (built & tested in the old session)

### 1) CSS — add to the stylesheet
Depends on these CSS vars (define equivalents if absent): `--line`, `--ink`, `--ink-soft`, `--brass`, `--forest`, `--shadow`. Fonts used: `Fraunces` (headings).

```css
  /* ---- opportunities / project cards ---- */
  .opp-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:26px;margin-top:56px}
  .pcard{background:#fff;border:1px solid var(--line);border-radius:18px;overflow:hidden;display:flex;flex-direction:column;transition:.35s;box-shadow:0 1px 2px rgba(22,32,26,.04)}
  .pcard:hover{transform:translateY(-6px);box-shadow:var(--shadow)}
  .pcard-img{position:relative;aspect-ratio:16/11;overflow:hidden}
  .pcard-img>img{width:100%;height:100%;object-fit:cover;display:block}
  .pcard-img.ph-1{background:linear-gradient(135deg,#2c4a3a,#1d352a)}
  .pcard-img.ph-2{background:linear-gradient(135deg,#324352,#1f2d36)}
  .pcard-img.ph-3{background:linear-gradient(135deg,#7a5a32,#b08948)}
  .pcard-img.ph-4{background:linear-gradient(135deg,#3a5446,#23362c)}
  .pcard-img.ph-5{background:linear-gradient(135deg,#574a6a,#2e2740)}
  .pill{position:absolute;top:15px;display:inline-flex;align-items:center;gap:8px;font-size:.76rem;font-weight:600;padding:9px 15px;border-radius:100px;line-height:1}
  .pill-status{left:15px;color:#fff}
  .st-uc{background:var(--brass)}
  .st-pc{background:var(--forest)}
  .st-cp{background:#2c7a4b}
  .pill-sale{right:15px;background:#fff;color:var(--ink);text-transform:uppercase;letter-spacing:.06em;font-size:.7rem;box-shadow:0 6px 16px -6px rgba(22,32,26,.4)}
  .pill-sale .dot{width:9px;height:9px;border-radius:50%;background:#37c871}
  .pill-sale.soon .dot{background:var(--brass)}
  .pill-sale.sold .dot{background:#c0392b}
  .pcard-body{padding:26px 26px 28px;display:flex;flex-direction:column;flex:1}
  .pcard-body h3{font-family:'Fraunces',serif;font-size:1.42rem;font-weight:600;line-height:1.14;color:var(--ink)}
  .pcard-addr{color:var(--ink-soft);font-size:.92rem;margin-top:8px}
  .pcard-desc{color:var(--ink);font-size:.95rem;margin-top:14px;line-height:1.55;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden}
  .pcard-prices{margin-top:auto;padding-top:18px;border-top:1px solid var(--line);display:flex;flex-direction:column;gap:13px}
  .price-row{display:flex;align-items:center;justify-content:space-between;gap:12px}
  .price-specs{display:flex;align-items:center;gap:15px;color:var(--ink-soft);font-size:.92rem}
  .price-specs .spec{display:inline-flex;align-items:center;gap:6px}
  .price-specs svg{width:19px;height:19px;flex-shrink:0}
  .price-amt{font-weight:700;font-size:1rem;color:var(--ink);white-space:nowrap}
  .price-amt small{font-weight:500;color:var(--ink-soft);font-size:.78rem;margin-right:4px}
  .price-tba{font-size:.92rem;color:var(--ink-soft);padding:4px 0}
  .price-tba a{color:var(--brass);font-weight:600}
```

### 2) HTML — the cards render into this container
Replace the old projects grid markup with just:
```html
    <div class="opp-grid" id="projGrid"></div>
```

### 3) JS — data + renderer (run after DOM is ready, before any reveal/IntersectionObserver)
To use real photos: set each project's `image` to the committed path, e.g. `Projects/Aluna/hero.jpg`. If `image` is set it replaces the gradient; otherwise the `ph-N` gradient placeholder shows.

```javascript
  /* ===================== PROJECT CARDS =====================
     Data-driven cards. To add / update a project from a price list:
       1) add an object to PROJECTS below,
       2) paste every price-list row into `listings` as {beds,baths,cars,price}
          (raw rows — duplicates fine; cars optional / 0 if none).
     Renderer groups rows into segments (phân khúc = beds-baths-cars) and shows
     ONE row per segment at the LOWEST available price ("from $…").
     status: under-construction | pre-construction | completed  (standardized)
     sale:   selling | coming-soon | sold-out                                  */
  const STATUS={
    'under-construction':{label:'Under construction',cls:'st-uc'},
    'pre-construction':  {label:'Pre-construction',  cls:'st-pc'},
    'completed':         {label:'Completed',         cls:'st-cp'}
  };
  const SALE={
    'selling':    {label:'Selling now', cls:''},
    'coming-soon':{label:'Coming soon', cls:'soon'},
    'sold-out':   {label:'Sold out',    cls:'sold'}
  };
  const ICON={
    bed:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M2 20v-8a2 2 0 0 1 2-2h16a2 2 0 0 1 2 2v8"/><path d="M4 10V6a2 2 0 0 1 2-2h12a2 2 0 0 1 2 2v4"/><path d="M2 18h20"/><path d="M12 4v6"/></svg>',
    bath:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="m4 4 2.5 2.5"/><path d="M13.5 6.5a4.95 4.95 0 0 0-7 7"/><path d="M15 5 5 15"/><path d="M14 17v.01"/><path d="M10 16v.01"/><path d="M13 13v.01"/><path d="M16 10v.01"/><path d="M11 20v.01"/><path d="M17 14v.01"/><path d="M20 11v.01"/></svg>',
    car:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M19 17h2c.6 0 1-.4 1-1v-3c0-.9-.7-1.7-1.5-1.9C18.7 10.6 16 10 16 10s-1.3-1.4-2.2-2.3c-.5-.4-1.1-.7-1.8-.7H5c-.6 0-1.1.4-1.4.9l-1.4 2.9A3.7 3.7 0 0 0 2 12v4c0 .6.4 1 1 1h2"/><circle cx="7" cy="17" r="2"/><path d="M9 17h6"/><circle cx="17" cy="17" r="2"/></svg>'
  };
  const money=n=>'$'+Number(n).toLocaleString('en-US');

  // Real Rivera projects. Set `image` to a committed photo path to replace the gradient.
  const PROJECTS=[
    {name:'Aluna at Collins Wharf', address:'989 Collins Street, Docklands VIC 3008',
     status:'pre-construction', sale:'selling', img:'ph-1', image:'',
     desc:'Bộ sưu tập căn hộ ven sông cuối cùng tại Collins Wharf của Lendlease, bên bờ Yarra với tầm nhìn nước và thành phố — hồ bơi trong nhà, gym, sauna và concierge.',
     listings:[]},
    {name:'Ancora at Collins Wharf', address:'971 Collins Street, Docklands VIC 3008',
     status:'under-construction', sale:'selling', img:'ph-2', image:'',
     desc:'Tháp 28 tầng, 303 dinh thự bên Victoria Harbour của Lendlease, kết nối tiện ích wellness cùng Regatta — hồ bơi trong nhà, spa, sauna và sảnh lễ tân kiểu khách sạn.',
     listings:[]},
    {name:'380 Melbourne', address:'371 Little Lonsdale Street, Melbourne VIC 3000',
     status:'completed', sale:'selling', img:'ph-3', image:'',
     desc:'Cặp tháp đôi 65 tầng cao 218m do Elenberg Fraser thiết kế — 728 căn hộ cùng khu bán lẻ và suite khách sạn ngay trung tâm Melbourne CBD.',
     listings:[]},
    {name:'Aspire Melbourne', address:'299 King Street, Melbourne VIC 3000',
     status:'completed', sale:'selling', img:'ph-4', image:'',
     desc:'Tháp 65 tầng do ICD Property phát triển, Elenberg Fraser thiết kế — 594 căn hộ với hồ bơi trong nhà, gym, ballroom và tầm nhìn toàn cảnh CBD.',
     listings:[]},
    {name:'671 Chapel Street, South Yarra', address:'671 Chapel Street, South Yarra VIC 3141',
     status:'under-construction', sale:'selling', img:'ph-5', image:'',
     desc:'20 tầng, 126 dinh thự cao cấp do Bates Smart thiết kế tại Como Precinct — wellness center, hồ bơi, concierge và tầm nhìn Yarra & Dandenong.',
     listings:[]}
  ];

  function renderProjects(){
    const grid=document.getElementById('projGrid');
    if(!grid) return;
    grid.innerHTML=PROJECTS.map(p=>{
      const st=STATUS[p.status]||STATUS['pre-construction'];
      const sl=SALE[p.sale]||SALE['selling'];
      // group price-list rows by segment, keep the lowest price per segment
      const segs=new Map();
      (p.listings||[]).forEach(l=>{
        const cars=l.cars||0, key=l.beds+'-'+l.baths+'-'+cars;
        const cur=segs.get(key);
        if(!cur) segs.set(key,{beds:l.beds,baths:l.baths,cars:cars,price:l.price});
        else if(l.price<cur.price) cur.price=l.price;
      });
      const rows=[...segs.values()].sort((a,b)=>a.beds-b.beds||a.baths-b.baths||a.cars-b.cars||a.price-b.price);
      const prices=rows.length
        ? rows.map(s=>{
            const specs='<span class="spec">'+ICON.bed+s.beds+'</span>'+
                        '<span class="spec">'+ICON.bath+s.baths+'</span>'+
                        (s.cars?'<span class="spec">'+ICON.car+s.cars+'</span>':'');
            return '<div class="price-row"><div class="price-specs">'+specs+'</div>'+
                   '<div class="price-amt"><small>from</small>'+money(s.price)+'</div></div>';
          }).join('')
        : '<div class="price-tba">Bảng giá: <a href="#contact">liên hệ Rivera</a></div>';
      const media=p.image?'<img src="'+p.image+'" alt="'+p.name+'">':'';
      return '<article class="pcard reveal">'+
        '<div class="pcard-img '+(p.img||'')+'">'+media+
          '<span class="pill pill-status '+st.cls+'">'+st.label+'</span>'+
          '<span class="pill pill-sale '+sl.cls+'"><span class="dot"></span>'+sl.label+'</span>'+
        '</div>'+
        '<div class="pcard-body">'+
          '<h3>'+p.name+'</h3>'+
          '<div class="pcard-addr">'+p.address+'</div>'+
          '<p class="pcard-desc">'+p.desc+'</p>'+
          '<div class="pcard-prices">'+prices+'</div>'+
        '</div>'+
      '</article>';
    }).join('');
  }
  renderProjects();
```

---

## First prompt to paste into the new (rivera-website) session
> Continuing project-card work from a handover. Repo is `rivera-website`. Read HANDOVER.md (pasted below / committed). Tasks: (1) apply the project-card rebuild — standardized top-left status pill, top-right sales pill, title/address/2-line desc, and a pricing block showing the LOWEST price per beds-baths-cars segment as "from $X"; (2) the `Projects/` photo folders are committed in the repo — browse each, pick the best hero shot, wire each card's image; (3) I'll paste a price list per project for you to extract into `listings`. Adapt the markup to this repo's structure but keep the format, the 3-value status taxonomy, on-brand pill colours, and the pricing rule identical.
