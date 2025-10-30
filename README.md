import json
import yaml
import io
import datetime as dt
from dataclasses import dataclass, asdict
from typing import List, Dict
import pandas as pd
import streamlit as st

# ---------- Load data ----------
@st.cache_data
def load_questions(path: str = "questions.json") -> List[Dict]:
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

@st.cache_data
def load_rules(path: str = "feedback_rules.yaml") -> Dict:
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)

# ---------- Feedback engine ----------
def generate_feedback(question_text: str, answer_text: str, rules: Dict, lang: str) -> str:
    # ルールは「含まれるキーワードで最初にマッチしたもの」を返すシンプル仕様
    for rule in rules.get("rules", []):
        if any(key in question_text for key in rule.get("keywords", [])):
            return rule["feedback"].get(lang, rule["feedback"].get("ja", ""))
    # 既定（その他）
    return rules["default"]["feedback"].get(lang, rules["default"]["feedback"]["ja"])

# ---------- Data model ----------
@dataclass
class QAItem:
    q_ja: str
    q_en: str
    answer: str = ""
    feedback_ja: str = ""
    feedback_en: str = ""

# ---------- UI helpers ----------
def init_state(questions_raw):
    if "lang" not in st.session_state:
        st.session_state.lang = "ja"  # 'ja' or 'en'
    if "index" not in st.session_state:
        st.session_state.index = 0
    if "qa" not in st.session_state:
        st.session_state.qa: List[QAItem] = [
            QAItem(q_ja=q["ja"], q_en=q["en"]) for q in questions_raw
        ]
    if "timestamp" not in st.session_state:
        st.session_state.timestamp = dt.datetime.now().strftime("%Y-%m-%d_%H%M%S")

def header():
    st.title("就活準備チャットボット / Job Hunting Prep Chatbot")
    st.caption("やさしい自動フィードバックつき｜AI-assisted gentle feedback")

def lang_toggle():
    col1, col2 = st.columns([1,3])
    with col1:
        st.session_state.lang = st.segmented_control(
            "Language / 言語", options=["ja", "en"], default=st.session_state.lang
        )
    with col2:
        st.write("")

def show_progress():
    total = len(st.session_state.qa)
    idx = st.session_state.index
    st.progress((idx)/total if total else 0)
    st.caption(f"Progress: {idx}/{total} answered")

def ask_current_question(rules):
    idx = st.session_state.index
    qa_item = st.session_state.qa[idx]
    lang = st.session_state.lang

    question = qa_item.q_ja if lang == "ja" else qa_item.q_en
    st.subheader(f"Q{idx+1}")
    st.write(question)

    placeholder = "ここに入力してください / Type your answer here..."
    qa_item.answer = st.text_area("A:", value=qa_item.answer, height=140, placeholder=placeholder)

    # フィードバック生成
    fb_ja = generate_feedback(qa_item.q_ja, qa_item.answer, rules, "ja")
    fb_en = generate_feedback(qa_item.q_ja, qa_item.answer, rules, "en")

    qa_item.feedback_ja = fb_ja
    qa_item.feedback_en = fb_en

    st.divider()
    with st.expander("やさしいフィードバック / Gentle Feedback", expanded=True):
        colA, colB = st.columns(2)
        with colA:
            st.markdown("**日本語**")
            st.write(fb_ja)
        with colB:
            st.markdown("**English**")
            st.write(fb_en)

def nav_buttons():
    idx = st.session_state.index
    n = len(st.session_state.qa)
    colL, colM, colR = st.columns([1,1,1])

    with colL:
        if st.button("← 前へ / Prev", disabled=(idx == 0)):
            st.session_state.index -= 1
            st.rerun()

    with colM:
        if st.button("次へ → / Next", disabled=(idx >= n-1)):
            st.session_state.index += 1
            st.rerun()

    with colR:
        if st.button("結果を見る / See Summary", type="primary", disabled=(idx < n-1)):
            st.session_state.index = n
            st.rerun()

def to_frames() -> pd.DataFrame:
    rows = []
    for i, item in enumerate(st.session_state.qa, start=1):
        rows.append({
            "index": i,
            "question_ja": item.q_ja,
            "question_en": item.q_en,
            "answer": item.answer,
            "feedback_ja": item.feedback_ja,
            "feedback_en": item.feedback_en,
        })
    return pd.DataFrame(rows)

def downloads(df: pd.DataFrame):
    stamp = st.session_state.timestamp
    csv_bytes = df.to_csv(index=False).encode("utf-8-sig")
    json_bytes = df.to_json(orient="records", force_ascii=False, indent=2).encode("utf-8")

    st.download_button("CSVをダウンロード / Download CSV",
                       data=csv_bytes, file_name=f"profile_summary_{stamp}.csv", mime="text/csv")
    st.download_button("JSONをダウンロード / Download JSON",
                       data=json_bytes, file_name=f"profile_summary_{stamp}.json", mime="application/json")

def summary_page():
    st.success("回答がすべて揃いました / All questions answered!")
    df = to_frames()
    st.dataframe(df, use_container_width=True, hide_index=True)
    downloads(df)

    st.divider()
    st.subheader("再スタート / Start Over")
    if st.button("回答をリセット / Reset"):
        # すべてリセット
        questions_raw = load_questions()
        st.session_state.clear()
        init_state(questions_raw)
        st.rerun()

# ---------- App main ----------
def main():
    st.set_page_config(page_title="Job Interview Prep", page_icon="💼", layout="centered")
    questions_raw = load_questions()
    rules = load_rules()
    init_state(questions_raw)

    header()
    lang_toggle()
    show_progress()

    if st.session_state.index < len(st.session_state.qa):
        ask_current_question(rules)
        nav_buttons()
    else:
        summary_page()

if __name__ == "__main__":
    main()
