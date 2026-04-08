import streamlit as st
import urllib.parse

# ─────────────────────────────────────────────
# Page config
# ─────────────────────────────────────────────
st.set_page_config(
    page_title="Overpass Turbo – Protection Infrastructures",
    page_icon="⚡",
    layout="wide",
)

# ─────────────────────────────────────────────
# Custom CSS
# ─────────────────────────────────────────────
st.markdown("""
<style>
    .block-container { max-width: 1200px; }
    .stCodeBlock { border: 1px solid #444; border-radius: 8px; }
    div[data-testid="stExpander"] details {
        border: 1px solid #555; border-radius: 8px;
    }
    h1 { text-align: center; }
    .info-box {
        background: #1a1a2e; color: #e0e0e0;
        padding: 12px 16px; border-radius: 8px;
        border-left: 4px solid #e94560; margin-bottom: 16px;
        font-size: 0.9rem;
    }
</style>
""", unsafe_allow_html=True)

# ─────────────────────────────────────────────
# Title
# ─────────────────────────────────────────────
st.title("⚡ Générateur de requêtes Overpass Turbo")
st.markdown(
    "<p style='text-align:center;color:gray;'>"
    "Protection des infrastructures – Pylônes RTE</p>",
    unsafe_allow_html=True,
)
st.divider()

# ═══════════════════════════════════════════════
# SIDEBAR – All parameters
# ═══════════════════════════════════════════════
with st.sidebar:
    st.header("⚙️ Paramètres de la requête")

    # ── Global ──────────────────────────────
    st.subheader("🌐 Général")
    timeout = st.slider("Timeout (secondes)", 30, 600, 180, step=30)
    output_format = st.selectbox("Format de sortie", ["json", "xml", "csv"], index=0)
    operator_pattern = st.text_input(
        "Opérateur (regex)", value="RTE", help="Expression régulière pour filtrer l'opérateur des pylônes et lignes."
    )
    case_insensitive_operator = st.checkbox("Insensible à la casse (opérateur)", value=True)

    st.divider()

    # ── 1) Chemins / routes légères ────────
    st.subheader("1️⃣ Chemins & petites routes")

    all_path_types = [
        "track", "path", "footway", "unclassified",
        "service", "tertiary", "secondary", "pedestrian",
        "cycleway", "bridleway", "living_street", "residential",
    ]
    default_path_types = [
        "track", "path", "footway", "unclassified",
        "service", "tertiary", "secondary", "pedestrian",
    ]
    selected_paths = st.multiselect(
        "Types de voies (highway)",
        options=all_path_types,
        default=default_path_types,
        help="Catégories highway utilisées pour la proximité aux pylônes.",
    )
    include_admin_boundaries = st.checkbox(
        "Inclure limites administratives (boundary=administrative)", value=True
    )

    st.divider()

    # ── 2) Pylônes RTE ─────────────────────
    st.subheader("2️⃣ Pylônes RTE – proximité chemins")
    tower_path_distance = st.slider(
        "Distance max pylône ↔ chemin (m)", 10, 500, 60, step=5
    )
    power_element = st.selectbox(
        "Élément power", ["tower", "pole", "portal", "terminal"], index=0
    )

    st.divider()

    # ── 3) Bâtiments pertinents ─────────────
    st.subheader("3️⃣ Bâtiments pertinents")
    enable_buildings = st.checkbox("Activer le filtre bâtiments", value=True)
    building_distance = st.slider(
        "Distance max pylône ↔ bâtiment (m)", 50, 2000, 400, step=50,
        disabled=not enable_buildings,
    )
    exclude_wall_no = st.checkbox(
        'Exclure wall=no', value=True, disabled=not enable_buildings
    )
    exclude_cadastre = st.checkbox(
        'Exclure source~"cadastre-dgi-fr"', value=True,
        disabled=not enable_buildings,
    )
    custom_source_exclude = st.text_input(
        "Source à exclure (regex, optionnel)", value="",
        disabled=not enable_buildings,
        help="Regex supplémentaire pour exclure des sources de bâtiments.",
    )

    st.divider()

    # ── 5-6) Routes principales ─────────────
    st.subheader("5️⃣ Routes principales")
    enable_main_roads = st.checkbox("Activer le filtre routes principales", value=True)
    all_road_types = [
        "motorway", "trunk", "primary", "secondary", "tertiary",
        "motorway_link", "trunk_link", "primary_link",
    ]
    default_road_types = ["motorway", "trunk", "primary", "secondary", "tertiary"]
    selected_roads = st.multiselect(
        "Types de routes principales",
        options=all_road_types,
        default=default_road_types,
        disabled=not enable_main_roads,
    )
    tower_road_distance = st.slider(
        "Distance max pylône ↔ route principale (m)", 10, 500, 60, step=5,
        disabled=not enable_main_roads,
    )

    st.divider()

    # ── 8) Lignes RTE ──────────────────────
    st.subheader("8️⃣ Lignes électriques RTE")
    include_rte_lines = st.checkbox("Inclure les lignes RTE dans la sortie", value=True)

    st.divider()

    # ── Sortie ──────────────────────────────
    st.subheader("📤 Options de sortie")
    out_body = st.checkbox("out body", value=True)
    out_skel = st.checkbox("out skel qt", value=True)
    out_geom = st.checkbox("out geom (géométrie complète)", value=False)


# ═══════════════════════════════════════════════
# QUERY BUILDER
# ═══════════════════════════════════════════════
def build_query() -> str:
    ci = ",i" if case_insensitive_operator else ""
    lines: list[str] = []

    # Header
    lines.append(f"[out:{output_format}][timeout:{timeout}];")
    lines.append("")

    # ── Step 1: Paths ──────────────────────
    if selected_paths or include_admin_boundaries:
        lines.append("// 1) Chemins + petites routes + limites administratives")
        lines.append("(")
        if selected_paths:
            regex = "^(" + "|".join(selected_paths) + ")$"
            lines.append(f'  way')
            lines.append(f'    ["highway"~"{regex}"]')
            lines.append(f'    ({{{{bbox}}}});')
        if include_admin_boundaries:
            lines.append(f'  way')
            lines.append(f'    ["boundary"="administrative"]')
            lines.append(f'    ({{{{bbox}}}});')
        lines.append(")->.paths;")
        lines.append("")

    # ── Step 2: Towers near paths ──────────
    lines.append(f"// 2) Pylônes {operator_pattern} à moins de {tower_path_distance} m des chemins")
    lines.append("node")
    lines.append(f'  ["power"="{power_element}"]')
    lines.append(f'  ["operator"~"{operator_pattern}"{ci}]')
    lines.append(f"  (around.paths:{tower_path_distance})")
    lines.append(f"  ->.rte_towers_near_paths;")
    lines.append("")

    # ── Step 3: Buildings ──────────────────
    if enable_buildings:
        lines.append(f"// 3) Bâtiments pertinents à moins de {building_distance} m des pylônes")
        lines.append("way")
        lines.append('  ["building"]')
        if exclude_wall_no:
            lines.append('  ["wall"!="no"]')
        if exclude_cadastre:
            lines.append('  ["source"!~"cadastre-dgi-fr",i]')
        if custom_source_exclude:
            lines.append(f'  ["source"!~"{custom_source_exclude}",i]')
        lines.append(f"  (around.rte_towers_near_paths:{building_distance})")
        lines.append("  ->.buildings;")
        lines.append("")

        # ── Step 4: Towers near buildings ──
        lines.append(f"// 4) Pylônes à moins de {building_distance} m d'un bâtiment pertinent")
        lines.append(f"node.rte_towers_near_paths(around.buildings:{building_distance})->.towers_near_buildings;")
        lines.append("")

    # ── Step 5-6: Main roads ───────────────
    if enable_main_roads and selected_roads:
        road_regex = "^(" + "|".join(selected_roads) + ")$"
        lines.append("// 5) Routes principales")
        lines.append("way")
        lines.append(f'  ["highway"~"{road_regex}"]')
        lines.append(f"  ({{{{bbox}}}})")
        lines.append("  ->.main_roads;")
        lines.append("")

        lines.append(f"// 6) Pylônes à moins de {tower_road_distance} m d'une route principale")
        lines.append(f"node.rte_towers_near_paths(around.main_roads:{tower_road_distance})->.towers_near_main;")
        lines.append("")

    # ── Step 7: Exclusion & difference ─────
    has_exclusions = enable_buildings or (enable_main_roads and selected_roads)
    if has_exclusions:
        lines.append("// 7) Pylônes conservés = proches d'un chemin MAIS exclus si :")
        if enable_buildings:
            lines.append(f"//    - bâtiment pertinent à < {building_distance} m")
        if enable_main_roads and selected_roads:
            lines.append(f"//    - route principale à < {tower_road_distance} m")
        lines.append("")

        # 7a – union of excluded
        exclusion_sets = []
        if enable_buildings:
            exclusion_sets.append("  .towers_near_buildings;")
        if enable_main_roads and selected_roads:
            exclusion_sets.append("  .towers_near_main;")

        lines.append("// 7a) Union des pylônes à exclure")
        lines.append("(")
        lines.extend(exclusion_sets)
        lines.append(")->.towers_excluded;")
        lines.append("")

        # 7b – difference
        lines.append("// 7b) Différence")
        lines.append("(")
        lines.append("  .rte_towers_near_paths;")
        lines.append("  - .towers_excluded;")
        lines.append(")->.result;")
        lines.append("")
    else:
        # No exclusions → result = all towers near paths
        lines.append("// Pas de filtre d'exclusion → tous les pylônes proches de chemins")
        lines.append("(.rte_towers_near_paths;)->.result;")
        lines.append("")

    # ── Step 8: RTE lines ──────────────────
    if include_rte_lines:
        lines.append("// 8) Lignes électriques RTE")
        lines.append("way")
        lines.append(f'  ["power"="line"]')
        lines.append(f'  ["operator"~"{operator_pattern}"{ci}]')
        lines.append(f"  ({{{{bbox}}}})")
        lines.append("  ->.rte_lines;")
        lines.append("")

    # ── Step 9: Output ─────────────────────
    lines.append("// 9) Sortie")
    output_sets = [".result;"]
    if include_rte_lines:
        output_sets.append(".rte_lines;")
    lines.append("(")
    for s in output_sets:
        lines.append(f"  {s}")
    lines.append(");")

    out_parts = []
    if out_body:
        out_parts.append("out body;")
    if out_geom:
        out_parts.append("out geom;")
    out_parts.append(">;")
    if out_skel:
        out_parts.append("out skel qt;")

    lines.append(" ".join(out_parts))

    return "\n".join(lines)


# ═══════════════════════════════════════════════
# MAIN AREA – Display query
# ═══════════════════════════════════════════════
query = build_query()

col_preview, col_info = st.columns([3, 1])

with col_preview:
    st.subheader("📝 Requête générée")
    st.code(query, language="text", line_numbers=True)

    # ── Overpass Turbo link ────────────────
    encoded = urllib.parse.quote(query, safe="")
    overpass_url = f"https://overpass-turbo.eu/?Q={encoded}&R"
    st.markdown(
        f'<a href="{overpass_url}" target="_blank">'
        f'<button style="background:#e94560;color:white;border:none;'
        f'padding:10px 24px;border-radius:6px;font-size:1rem;cursor:pointer;">'
        f'🚀 Ouvrir dans Overpass Turbo</button></a>',
        unsafe_allow_html=True,
    )

with col_info:
    st.subheader("ℹ️ Résumé")

    active_filters = []
    if selected_paths:
        active_filters.append(f"**Chemins** : {len(selected_paths)} types")
    if include_admin_boundaries:
        active_filters.append("**Limites admin** : ✅")
    active_filters.append(f"**Pylônes** : `{power_element}` ({operator_pattern})")
    active_filters.append(f"**Dist. chemin** : {tower_path_distance} m")
    if enable_buildings:
        active_filters.append(f"**Bâtiments** : excl. < {building_distance} m")
    else:
        active_filters.append("**Bâtiments** : ❌ désactivé")
    if enable_main_roads and selected_roads:
        active_filters.append(f"**Routes princ.** : excl. < {tower_road_distance} m")
    else:
        active_filters.append("**Routes princ.** : ❌ désactivé")
    if include_rte_lines:
        active_filters.append("**Lignes RTE** : ✅")

    for f in active_filters:
        st.markdown(f"- {f}")

    st.divider()
    st.caption(f"Longueur requête : {len(query)} caractères")


# ═══════════════════════════════════════════════
# EXPLANATIONS (collapsible)
# ═══════════════════════════════════════════════
st.divider()
with st.expander("📖 Documentation de la requête", expanded=False):
    st.markdown("""
### Logique de la requête

La requête identifie les **pylônes électriques RTE potentiellement vulnérables** en suivant cette logique :

| Étape | Description |
|-------|-------------|
| **1** | Sélection des chemins d'accès (petites routes, sentiers) et limites administratives dans la bbox |
| **2** | Identification des pylônes RTE proches de ces chemins (= accessibles) |
| **3** | Recherche de bâtiments pertinents autour de ces pylônes (= zones habitées) |
| **4** | Identification des pylônes proches de bâtiments |
| **5** | Sélection des routes principales dans la bbox |
| **6** | Identification des pylônes proches de routes principales (= visibles) |
| **7** | **Exclusion** : on retire les pylônes proches de bâtiments ou routes principales |
| **8** | Ajout optionnel des lignes électriques RTE pour contexte visuel |
| **9** | Sortie finale |

### Résultat
Les pylônes restants sont ceux qui sont :
- ✅ **Accessibles** (proches d'un chemin / limite)
- ✅ **Isolés** (pas de bâtiment à proximité = pas de témoin)
- ✅ **Discrets** (pas proches d'une route principale = peu de passage)
    """)