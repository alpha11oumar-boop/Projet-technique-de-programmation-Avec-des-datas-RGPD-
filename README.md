# Projet-technique-de-programmation-Avec-des-datas-RGPD-
# Nom des membres : ALPHA OUMAR BAH, ALPHA OUMAR DIALLO et YAYE FATOU GNINGUE
Analyseur de données linkedin RGPD
import streamlit as st
import pandas as pd
import plotly.express as px

# --- FONCTIONS UTILITAIRES (Hors de la boucle principale) ---

def extract_city(title):
    """Extrait la ville à partir du titre du poste."""
    if not isinstance(title, str):
        return "Non spécifié"
    if '-' in title:
        return title.split('-')[-1].strip()
    return "Non spécifié"


def extract_contract_type(title):
    """Détermine le type de contrat (Stage, Alternance, Autre)."""
    if not isinstance(title, str):
        return "Autre"
    
    title_lower = title.lower()
    if 'stage' in title_lower:
        return 'Stage'
    elif 'alternance' in title_lower:
        return 'Alternance'
    return 'Autre'


def load_file(uploaded_file):
    """Charge un fichier Excel/CSV selon son extension."""
    try:
        if uploaded_file.name.endswith('.ods'):
            return pd.read_excel(uploaded_file, engine="odf")
        elif uploaded_file.name.endswith('.xlsx'):
            return pd.read_excel(uploaded_file)
        else:
            return pd.read_csv(uploaded_file)
    except Exception as e:
        st.error(f"Erreur de lecture : {e}")
        return pd.DataFrame()


def clean_application_data(df):
    """Nettoie et enrichit le DataFrame des candidatures."""
    # Nettoyage des noms de colonnes
    df.columns = [c.strip() for c in df.columns]
    cols = df.columns.tolist()

    # Renommage standard
    if 'Reponse' in cols:
        df.rename(columns={'Reponse': 'Statut'}, inplace=True)
    if 'Lien_Offre' in cols:
        df.rename(columns={'Lien_Offre': 'Job Url'}, inplace=True)
    
    # Gestion du statut par défaut
    if 'Statut' not in df.columns:
        df['Statut'] = 'En attente'
    
    df['Statut'] = df['Statut'].fillna('En attente').replace('', 'En attente')
    
    # Gestion des dates
    col_date = 'Date' if 'Date' in cols else 'Application Date'
    
    # Conversion date avec gestion d'erreurs
    df['Date'] = pd.to_datetime(
        df[col_date], 
        format="%m/%d/%y, %I:%M %p", 
        errors='coerce'
    )
    df['Mois'] = df['Date'].dt.strftime('%Y-%m')
    
    # Feature Engineering (Appel des fonctions externes)
    df['Ville'] = df['Job Title'].apply(extract_city)
    df['Contrat'] = df['Job Title'].apply(extract_contract_type)
    
    return df


# --- FONCTION PRINCIPALE ---

def main():
    st.set_page_config(page_title="Projet RGPD Complet", layout="wide")
    st.title(" Analyseur de Données LinkedIn (Version Finale)")

    # --- 1. AIDE ORALE ---
    with st.expander(" NOTES POUR LA SOUTENANCE"):
        st.write("""
        * **Structure :** Code modulaire respectant les conditions de traitement.
        * **Technique :** Utilisation de Pandas pour le nettoyage et Plotly pour la visualisation.
        * **Sécurité :** Traitement local sans base de données externe.
        """)

    # --- 2. SIDEBAR : IMPORTATION ---
    st.sidebar.header(" Importez vos fichiers")
    uploaded_files = st.sidebar.file_uploader(
        "Glissez vos fichiers ici", 
        accept_multiple_files=True
    )
    
    data_store = {}

    if uploaded_files:
        for file in uploaded_files:
            # Chargement générique
            df_raw = load_file(file)
            
            if df_raw.empty:
                continue

            # Identification du type de fichier
            cols = [c.strip() for c in df_raw.columns]
            
            if 'Application Date' in cols or 'Date' in cols:
                # C'est le fichier principal
                df_clean = clean_application_data(df_raw)
                data_store['APPS'] = df_clean
                st.sidebar.success(f" Candidatures ({len(df_clean)})")
                
            elif 'Saved Date' in cols:
                data_store['SAVED'] = df_raw
                st.sidebar.success(" Veille (Articles sauvegardés)")
                
            elif 'Job Titles' in cols:
                data_store['PREFS'] = df_raw
                st.sidebar.success(" Préférences emploi")

    if not data_store:
        st.info(" Veuillez charger vos fichiers exports pour commencer.")
        return

    # --- 3. DASHBOARD ---
    
    if 'APPS' in data_store:
        df = data_store['APPS']
        
        st.header("1. Analyse de la Performance")
        
        # --- KPIs ---
        total = len(df)
        nb_refus = len(df[df['Statut'] == 'Refus'])
        if total > 0:
            taux_refus = (nb_refus / total) * 100
        else:
            taux_refus = 0
            
        col1, col2, col3 = st.columns(3)
        col1.metric("Total Candidatures", total)
        col2.metric("Refus Déclarés", nb_refus)
        col3.metric("Taux de Refus", f"{taux_refus:.1f}%")
        
        st.divider()
        
        # --- Graphiques Ligne 1 ---
        left_col, right_col = st.columns(2)
        
        with left_col:
            st.subheader("Répartition des Statuts")
            fig_statut = px.pie(
                df, 
                names='Statut', 
                hole=0.4, 
                color_discrete_sequence=px.colors.sequential.RdBu
            )
            st.plotly_chart(fig_statut, use_container_width=True)
            
        with right_col:
            st.subheader("Top 10 Entreprises")
            # Choix dynamique de la colonne nom
            col_ent = 'Company Name' if 'Company Name' in df.columns else 'Entreprise'
            top_companies = df[col_ent].value_counts().head(10)
            st.bar_chart(top_companies)

        st.divider()
        st.header("2. Analyse Avancée")

        # --- Graphiques Ligne 2 ---
        row2_col1, row2_col2 = st.columns(2)
        
        with row2_col1:
            st.subheader("Chronologie des envois")
            # Agrégation temporelle
            timeline = df.groupby('Mois').size().reset_index(name='Volume')
            fig_time = px.bar(
                timeline, 
                x='Mois', 
                y='Volume', 
                title="Volume par Mois"
            )
            st.plotly_chart(fig_time, use_container_width=True)
            
        with row2_col2:
            st.subheader("Réussite par Type de Contrat")
            fig_hist = px.histogram(
                df, 
                x='Contrat', 
                color='Statut', 
                barmode='group',
                title="Détail par contrat"
            )
            st.plotly_chart(fig_hist, use_container_width=True)
            
        # --- Données brutes ---
        with st.expander(" Voir le tableau détaillé"):
            cols_to_show = ['Date', 'Company Name', 'Job Title', 'Ville', 'Statut']
            # On ne garde que les colonnes qui existent vraiment
            final_cols = [c for c in cols_to_show if c in df.columns]
            st.dataframe(df[final_cols], use_container_width=True)

    # --- 4. ANALYSE STRATÉGIQUE (Bonus) ---
    if 'PREFS' in data_store:
        st.divider()
        st.header("3. Alignement Stratégique")
        # Exemple de traitement sécurisé
        raw_prefs = str(data_store['PREFS'].iloc[0].get('Job Titles', ''))
        targets = [t.strip() for t in raw_prefs.split('|') if t.strip()]
        
        st.success(f" Vos cibles identifiées : {', '.join(targets)}")


if __name__ == "__main__":
    main()
