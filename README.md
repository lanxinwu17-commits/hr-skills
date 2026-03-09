pip install streamlit pandas plotly openpyxl requests beautifulsoup4 jieba
streamlit run app.py
import io
import re
from datetime import datetime

import pandas as pd
import plotly.express as px
import requests
import streamlit as st
from bs4 import BeautifulSoup
import jieba


# ============= 工具函数 =============

def detect_column(df, candidates):
    """
    在 DataFrame 列名中，找到第一个匹配的列名。
    candidates: 可能的列名列表（中文/英文别名）
    """
    cols_lower = {c.lower(): c for c in df.columns}
    for name in candidates:
        key = str(name).lower()
        if key in cols_lower:
            return cols_lower[key]
    # 模糊匹配：包含关系
    for c in df.columns:
        for name in candidates:
            if name.lower() in str(c).lower():
                return c
    return None


def str_contains_any(series, patterns):
    """判断字符串列中是否包含任意一个模式（用于“通过/是”等模糊匹配）"""
    s = series.astype(str).fillna("")
    pattern = "|".join(patterns)
    return s.str.contains(pattern, case=False, regex=True)


# ============= 模块 1：招聘漏斗分析 =============

def recruitment_funnel_tab():
    st.subheader("招聘漏斗自动化分析")

    uploaded_file = st.file_uploader("上传包含招聘全流程数据的 Excel 文件", type=["xlsx", "xls"], key="recruit_file")

    st.markdown(
        "字段建议包含：`候选人姓名`、`岗位`、`投递日期`、`初筛状态`、`面试状态`、`Offer状态`、`是否入职`。"
    )

    if not uploaded_file:
        return

    try:
        df = pd.read_excel(uploaded_file)
    except Exception as e:
        st.error(f"读取 Excel 失败：{e}")
        return

    if df.empty:
        st.warning("表格为空，请检查文件内容。")
        return

    st.markdown("#### 原始数据预览")
    st.dataframe(df.head())

    # 智能匹配列
    col_name = detect_column(df, ["候选人姓名", "姓名", "name", "candidate"])
    col_position = detect_column(df, ["岗位", "职位", "position", "job"])
    col_apply_date = detect_column(df, ["投递日期", "投递时间", "apply_date"])
    col_screen = detect_column(df, ["初筛状态", "简历筛选", "screen", "初筛"])
    col_interview = detect_column(df, ["面试状态", "面试结果", "interview"])
    col_offer = detect_column(df, ["Offer状态", "offer状态", "offer"])
    col_onboard = detect_column(df, ["是否入职", "入职状态", "onboard", "join"])

    stage_info = {
        "投递": len(df),
    }

    # 初筛
    if col_screen:
        screened_pass = str_contains_any(df[col_screen], ["通过", "合格", "是", "YES", "Y"])
        stage_info["初筛通过"] = int(screened_pass.sum())
    # 面试
    if col_interview:
        interview_pass = str_contains_any(df[col_interview], ["通过", "合格", "录用", "推荐"])
        stage_info["面试通过"] = int(interview_pass.sum())
    # Offer
    if col_offer:
        offer_made = str_contains_any(df[col_offer], ["发放", "已发", "offer", "接受", "accept"])
        stage_info["Offer发放"] = int(offer_made.sum())
    # 入职
    if col_onboard:
        onboard_yes = str_contains_any(df[col_onboard], ["是", "已入职", "Y", "YES"])
        stage_info["入职"] = int(onboard_yes.sum())

    # 构建漏斗数据（只保留有值且>0的阶段，顺序固定）
    ordered_stages = ["投递", "初筛通过", "面试通过", "Offer发放", "入职"]
    funnel_stages = []
    for s in ordered_stages:
        if s in stage_info and stage_info[s] > 0:
            funnel_stages.append({"stage": s, "count": stage_info[s]})

    if len(funnel_stages) < 2:
        st.warning("可用于漏斗分析的阶段数量不足，请确认至少包含两个有效阶段（如投递、初筛、面试等）。")
        return

    # Plotly 漏斗图
    fig = px.funnel(
        funnel_stages,
        x="count",
        y="stage",
        title="招聘漏斗",
    )
    st.plotly_chart(fig, use_container_width=True)

    # 计算转化率 & 流失率
    analysis_rows = []
    for i in range(len(funnel_stages) - 1):
        s_from = funnel_stages[i]
        s_to = funnel_stages[i + 1]
        prev_count = s_from["count"]
        next_count = s_to["count"]
        if prev_count == 0:
            conv = 0.0
        else:
            conv = next_count / prev_count
        drop = 1 - conv
        analysis_rows.append(
            {
                "from": s_from["stage"],
                "to": s_to["stage"],
                "from_count": prev_count,
                "to_count": next_count,
                "conversion": conv,
                "drop": drop,
            }
        )

    st.markdown("#### 各环节转化情况")
    analysis_df = pd.DataFrame(
        [
            {
                "环节": f"{row['from']} → {row['to']}",
                "上一环节人数": row["from_count"],
                "下一环节人数": row["to_count"],
                "转化率": f"{row['conversion'] * 100:.1f}%",
                "流失率": f"{row['drop'] * 100:.1f}%",
            }
            for row in analysis_rows
        ]
    )
    st.dataframe(analysis_df)

    # 找到流失率最高的环节
    if analysis_rows:
        worst = max(analysis_rows, key=lambda r: r["drop"])
        total_applied = funnel_stages[0]["count"]
        final_stage = funnel_stages[-1]
        total_conv = final_stage["count"] / total_applied if total_applied else 0

        st.markdown("#### 自动生成分析结论")
        text = (
            f"本次共有 **{total_applied}** 名候选人进入招聘流程，最终在 **{final_stage['stage']}** 阶段人数为 "
            f"**{final_stage['count']}**，整体转化率约为 **{total_conv * 100:.1f}%**。\n\n"
            f"从各环节来看，当前流失率最高的环节为 **【{worst['from']} → {worst['to']}】**，"
            f"该环节由 **{worst['from_count']}** 人降至 **{worst['to_count']}** 人，"
            f"转化率约为 **{worst['conversion'] * 100:.1f}%**，流失率约为 **{worst['drop'] * 100:.1f}%**。\n\n"
            f"建议重点复盘该环节的评估标准、沟通话术及候选人体验，例如："
            f"是否存在筛选过严、信息不对称、Offer 竞争力不足等问题。"
        )
        st.write(text)


# ============= 模块 2：人事基础数据校验 & 加班计算 =============

def hr_data_tab():
    st.subheader("人事基础数据导入 & 自动矫正")

    uploaded_file = st.file_uploader("上传人事基础数据 Excel（考勤/薪资）", type=["xlsx", "xls"], key="hr_file")

    st.markdown(
        "建议包含字段：`工号`、`姓名`、`银行卡号`、`日期/年月`、`基本工资`、`加班开始时间`、`加班结束时间` 或 `加班时长`。"
    )

    if not uploaded_file:
        return

    try:
        df = pd.read_excel(uploaded_file)
    except Exception as e:
        st.error(f"读取 Excel 失败：{e}")
        return

    if df.empty:
        st.warning("表格为空，请检查文件内容。")
        return

    st.markdown("#### 原始数据预览")
    st.dataframe(df.head())

    # 列匹配
    col_emp_id = detect_column(df, ["工号", "员工编号", "employee_id"])
    col_name = detect_column(df, ["姓名", "name"])
    col_bank = detect_column(df, ["银行卡", "银行卡号", "bank_account"])
    col_date = detect_column(df, ["日期", "年月", "date", "month"])
    col_base_salary = detect_column(df, ["基本工资", "底薪", "base_salary"])
    col_ot_hours = detect_column(df, ["加班时长", "加班小时", "overtime_hours"])
    col_ot_start = detect_column(df, ["加班开始时间", "加班开始", "ot_start"])
    col_ot_end = detect_column(df, ["加班结束时间", "加班结束", "ot_end"])

    issues = []

    # 工号校验
    if col_emp_id:
        emp_series = df[col_emp_id].astype(str)
        for idx, v in emp_series.items():
            problems = []
            if pd.isna(v) or v.strip() == "" or v.lower() in ["nan", "none"]:
                problems.append("工号为空")
            if not re.fullmatch(r"[0-9A-Za-z\-]+", v.strip() if isinstance(v, str) else str(v)):
                problems.append("工号包含非法字符")
            if problems:
                issues.append(
                    {"行号": idx + 2, "字段": col_emp_id, "问题": "；".join(problems)}
                )
        # 重复工号
        dup = emp_series[emp_series.duplicated(keep=False)]
        for idx in dup.index:
            issues.append({"行号": idx + 2, "字段": col_emp_id, "问题": "工号重复"})

    # 银行卡校验
    if col_bank:
        bank_series = df[col_bank].astype(str).str.replace(" ", "")
        for idx, v in bank_series.items():
            problems = []
            if v == "" or v.lower() in ["nan", "none"]:
                problems.append("银行卡号为空")
            if not re.fullmatch(r"[0-9]{12,30}", v):
                problems.append("银行卡号长度/格式异常（需为 12-30 位数字）")
            if problems:
                issues.append(
                    {"行号": idx + 2, "字段": col_bank, "问题": "；".join(problems)}
                )

    # 日期解析
    if col_date:
        def parse_date(x):
            try:
                return pd.to_datetime(x)
            except Exception:
                return pd.NaT

        df["_parsed_date"] = df[col_date].apply(parse_date)
        for idx, v in df["_parsed_date"].items():
            if pd.isna(v):
                issues.append(
                    {"行号": idx + 2, "字段": col_date, "问题": "日期格式无法解析"}
                )

    # 计算加班时长
    if col_ot_hours:
        # 已有加班时长列，尽量转成数值
        df["_ot_hours"] = pd.to_numeric(df[col_ot_hours], errors="coerce")
    elif col_ot_start and col_ot_end:
        def calc_hours(row):
            try:
                start = pd.to_datetime(row[col_ot_start])
                end = pd.to_datetime(row[col_ot_end])
                return max((end - start).total_seconds() / 3600, 0)
            except Exception:
                return None

        df["_ot_hours"] = df.apply(calc_hours, axis=1)
    else:
        df["_ot_hours"] = 0.0

    # 标记异常加班时长
    for idx, v in df["_ot_hours"].items():
        if v is None or pd.isna(v):
            issues.append(
                {"行号": idx + 2, "字段": "加班时长", "问题": "加班时长缺失或无法计算"}
            )
        elif v < 0:
            issues.append(
                {"行号": idx + 2, "字段": "加班时长", "问题": "加班时长为负数"}
            )

    # 计算“每人每月总加班时长”和“有效加班时长（扣除前10h）”
    if col_emp_id and col_date:
        df["_month"] = df["_parsed_date"].dt.to_period("M").astype(str)
        group_cols = [col_emp_id, "_month"]
    elif col_emp_id:
        df["_month"] = "ALL"
        group_cols = [col_emp_id, "_month"]
    else:
        df["_month"] = "ALL"
        group_cols = ["_month"]

    monthly_ot = (
        df.groupby(group_cols)["_ot_hours"]
        .sum(min_count=1)
        .reset_index()
        .rename(columns={"_ot_hours": "total_ot_hours"})
    )
    monthly_ot["effective_ot_hours"] = monthly_ot["total_ot_hours"].apply(
        lambda x: max(x - 10, 0) if pd.notna(x) else 0
    )

    # 合并回原数据
    df = df.merge(monthly_ot, on=group_cols, how="left")

    # 计算加班工资
    st.markdown("#### 加班工资参数设置")
    default_rate = 1.5
    ot_rate = st.number_input(
        "加班工资系数（相对于日薪）", value=default_rate, step=0.1, min_value=0.0
    )

    if col_base_salary:
        # 简化算法：日薪 = 基本工资 / 21.75，小时工资 = 日薪 / 8
        base = pd.to_numeric(df[col_base_salary], errors="coerce")
        daily = base / 21.75
        hourly = daily / 8
        df["overtime_pay"] = df["effective_ot_hours"] * hourly * ot_rate
    else:
        df["overtime_pay"] = 0.0

    st.markdown("#### 矫正后数据")
    st.dataframe(df.head())

    # 问题明细表
    st.markdown("#### 数据问题明细")
    if issues:
        issues_df = pd.DataFrame(issues)
        st.dataframe(issues_df)
    else:
        st.success("未发现明显数据问题。")

    # 导出 Excel
    output = io.BytesIO()
    with pd.ExcelWriter(output, engine="openpyxl") as writer:
        df.to_excel(writer, index=False, sheet_name="矫正后数据")
        if issues:
            issues_df.to_excel(writer, index=False, sheet_name="数据问题明细")
    output.seek(0)

    st.download_button(
        label="下载矫正后数据（含问题明细）",
        data=output,
        file_name="hr_data_corrected.xlsx",
        mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    )


# ============= 模块 3：招聘文案 & Midjourney 提示词 =============

def extract_keywords_from_text(text, top_k=10):
    # 简单中文分词 + 词频统计
    words = [w for w in jieba.cut(text) if len(w.strip()) > 1]
    freq = {}
    for w in words:
        freq[w] = freq.get(w, 0) + 1
    sorted_items = sorted(freq.items(), key=lambda x: x[1], reverse=True)
    return [w for w, _ in sorted_items[:top_k]]


def fetch_site_info(url):
    try:
        resp = requests.get(url, timeout=8)
        resp.raise_for_status()
        html = resp.text
    except Exception:
        return None, None

    soup = BeautifulSoup(html, "html.parser")

    # 文本内容
    texts = " ".join(t.get_text(separator=" ", strip=True) for t in soup.find_all(["p", "h1", "h2", "h3"]))
    # 颜色（从 style / link rel=stylesheet 中简单抓取 #xxxxxx）
    css_texts = []
    for tag in soup.find_all(["style", "link"]):
        if tag.name == "style" and tag.string:
            css_texts.append(tag.string)
        elif tag.name == "link" and "stylesheet" in tag.get("rel", []):
            href = tag.get("href")
            if href and href.startswith("http"):
                try:
                    css_resp = requests.get(href, timeout=5)
                    css_resp.raise_for_status()
                    css_texts.append(css_resp.text)
                except Exception:
                    pass

    css_all = "\n".join(css_texts)
    colors = re.findall(r"#(?:[0-9a-fA-F]{3}){1,2}", css_all)
    color_counts = {}
    for c in colors:
        color_counts[c] = color_counts.get(c, 0) + 1
    main_colors = sorted(color_counts.items(), key=lambda x: x[1], reverse=True)
    main_colors = [c for c, _ in main_colors[:3]]

    return texts, main_colors


def copywriting_tab():
    st.subheader("招聘&企业文化活动文案 + Midjourney 提示词")

    url = st.text_input("企业官网 URL（可选）", placeholder="例如：https://www.example.com")
    company_intro = st.text_area("公司简介（可选）", height=150)

    style = st.radio(
        "选择视觉/文案风格",
        options=["专业黑金", "极简蓝白"],
        index=0,
    )

    if not url and not company_intro:
        st.info("请至少提供官网 URL 或一段公司简介。")
        return

    if st.button("生成文案与 Midjourney 提示词"):
        with st.spinner("正在分析企业信息并生成内容..."):
            texts = company_intro or ""
            main_colors = []

            if url:
                t, colors = fetch_site_info(url)
                if t:
                    texts += "\n" + t
                if colors:
                    main_colors = colors

            if not texts.strip():
                texts = "我们是一家专注于智能驾驶与自动驾驶领域的科技公司，聚焦高性能算法与车规级量产落地。"

            keywords = extract_keywords_from_text(texts, top_k=10)
            if not keywords:
                keywords = ["智能驾驶", "自动驾驶", "算法", "感知", "控制"]

            # 招聘文案生成
            kw_str = "、".join(keywords[:5])
            if style == "专业黑金":
                tone_intro = (
                    "在这里，我们以旗舰级技术实力与高标准工程实践，"
                    "推动智能驾驶从前沿探索走向大规模量产。"
                )
                tone_end = (
                    "如果你希望在高标准、快节奏、强协作的团队中，一起把代码写进未来的道路，"
                    "欢迎投递「智能驾驶算法工程师」岗位，与我们一起把想象变成量产。"
                )
                style_hint = "整体视觉建议采用黑金配色，强调高级感与科技质感。"
            else:  # 极简蓝白
                tone_intro = (
                    "在这里，我们用理性严谨的数据与工程实践，"
                    "一步步构建稳定可靠的自动驾驶系统。"
                )
                tone_end = (
                    "如果你期待在干净透明的工程环境中，将复杂算法落地到真实道路，"
                    "欢迎申请「智能驾驶算法工程师」，与我们以代码驱动每一次安全出行。"
                )
                style_hint = "整体视觉建议采用极简蓝白配色，强调清晰、理性与信任感。"

            copy = (
                f"【智能驾驶算法工程师招聘】\n\n"
                f"我们是一家专注于 {kw_str} 等领域的技术驱动型公司，"
                f"{tone_intro}\n\n"
                f"在智能驾驶算法工程师岗位上，你将深度参与感知、决策与控制等核心模块，"
                f"与算法、软件、硬件团队协同，推动自动驾驶系统在复杂道路场景中的稳定表现。\n\n"
                f"{tone_end}\n\n"
                f"{style_hint}"
            )

            st.markdown("#### 招聘推文（可用于社交平台/内推群）")
            st.write(copy)

            # Midjourney 提示词
            if style == "专业黑金":
                color_phrase = "black and gold, dark background, high contrast"
            else:
                color_phrase = "minimal blue and white, clean background, high contrast"

            base_prompts = [
                "futuristic autonomous driving control center, engineers analyzing algorithm visualizations, "
                "holographic dashboards, neural network flows, ultra-detailed, cinematic lighting, {color}, 16:9",
                "autonomous vehicle on highway at night, algorithm data overlay, lidar point clouds, "
                "clean UI, isometric style, {color}, 16:9",
                "AI chip and circuit board with flowing code and data streams, representing intelligent driving algorithms, "
                "macro shot, depth of field, {color}, 4:5",
                "team of engineers in modern tech office, large screens showing trajectory planning and perception maps, "
                "minimalist interface, {color}, 16:9",
                "conceptual scene of a city with autonomous cars, data lines connecting vehicles, "
                "futuristic but realistic, {color}, 21:9",
            ]

            st.markdown("#### Midjourney 提示词（英文，可直接复制）")
            for i, tmpl in enumerate(base_prompts, start=1):
                prompt = tmpl.format(color=color_phrase)
                st.code(prompt, language="text")

            if main_colors:
                st.markdown("#### 从官网解析到的主色（可作为设计参考）")
                st.write(", ".join(main_colors))


# ============= 主入口：Tab 布局 =============

def main():
    st.set_page_config(page_title="轻量化 HR 仪表盘", layout="wide")
    st.title("轻量化 HR 仪表盘")
    st.caption("招聘漏斗 · 人事数据校验 · 招聘文案 & Midjourney 提示词")

    tabs = st.tabs(["招聘漏斗分析", "人事基础数据 & 加班", "招聘文案 & MJ 提示词"])

    with tabs[0]:
        recruitment_funnel_tab()
    with tabs[1]:
        hr_data_tab()
    with tabs[2]:
        copywriting_tab()


if __name__ == "__main__":
    main()
