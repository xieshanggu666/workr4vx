<script setup>
import { computed, ref } from 'vue'
import { useHrStore } from '@/store/hr'

const store = useHrStore()
// 匹配度分布口径：snapshot=投递时历史评分；stage=进入当前阶段时评分；latest=按最新策略重算结果
const scoreView = ref('snapshot')
// 漏斗口径：current=按当前所处阶段；reached=按曾经到达（事件去重，回退后仍计入到达过）
const funnelMode = ref('current')

const STAGE_ORDER = ['submitted', 'screening', 'interview', 'offer', 'hired']
const STAGE_NAME = { submitted: '投递', screening: '筛选', interview: '面试', offer: 'Offer', hired: '录用' }

// 某应聘是否曾到达某阶段：正式事件（含回退事件）为准；hired 需 offer_accepted/hired 事件
function reachedStage(a, idx) {
  const target = STAGE_ORDER[idx]
  if (funnelMode.value === 'current') {
    const cur = STAGE_ORDER.indexOf(a.stage)
    return cur >= 0 && cur >= idx
  }
  // reached：事件中存在该阶段（回退后再次进入会刷新该阶段正式事件，仍然算到达）
  return (a.events || []).some(e => e.stage === target)
}

const funnel = computed(() =>
  STAGE_ORDER.map((s, idx) => {
    const count = store.applications.filter(a => reachedStage(a, idx)).length
    const prev = idx === 0 ? null : store.applications.filter(a => reachedStage(a, idx - 1)).length
    const conv = prev ? Math.round((count / prev) * 100) : null
    return { s, label: STAGE_NAME[s], count, conv }
  })
)

const overallConv = computed(() => {
  const sub = funnel.value[0]?.count || 0
  const hire = funnel.value[4]?.count || 0
  return sub ? Math.round(hire / sub * 100) : 0
})

const offerAcceptedCount = computed(() => store.offers.filter(o => ['accepted', 'joined'].includes(o.status)).length)
const joinedCount = computed(() => store.offers.filter(o => o.status === 'joined').length)
const pendingOfferCount = computed(() => store.offers.filter(o => o.status === 'pending').length)
// Offer→入职转化：口径为已入职 / 接受过 Offer
const offerJoinRate = computed(() => offerAcceptedCount.value
  ? Math.round(joinedCount.value / offerAcceptedCount.value * 100) : 0)

const channelCost = computed(() => {
  const map = {}
  store.candidates.forEach(c => map[c.channel] = (map[c.channel] || 0) + 1)
  const channelOfApp = a => store.candidates.find(c => c.id === a.candidate_id)?.channel
  return store.channels.map(ch => {
    const count = map[ch.name] || 0
    const joined = store.applications.filter(a => a.offer?.status === 'joined' && channelOfApp(a) === ch.name).length
    const accepted = store.applications.filter(a => ['accepted', 'joined'].includes(a.offer?.status) && channelOfApp(a) === ch.name).length
    return { name: ch.name, cost: ch.cost, count, cpc: count ? Math.round(ch.cost / count) : 0, joined, accepted }
  }).sort((a, b) => a.cpc - b.cpc)
})

const deptProgress = computed(() => {
  const map = {}
  store.applications.forEach(a => {
    const dept = a.dept
    map[dept] = map[dept] || { total: 0, hired: 0, joined: 0, offer: 0, interview: 0 }
    map[dept].total++
    if (a.offer?.status === 'joined') { map[dept].joined++; map[dept].hired++ }
    else if (a.stage === 'hired') map[dept].hired++
    if (a.stage === 'offer' || a.stage === 'hired') map[dept].offer++
    if (a.stage === 'interview' || a.stage === 'offer' || a.stage === 'hired') map[dept].interview++
  })
  return Object.entries(map).map(([k, v]) => ({ dept: k, ...v }))
})

const scoreOf = a => {
  if (scoreView.value === 'latest') return a.match?.score ?? a.stageSnapshot?.score ?? a.matchSnapshot?.score ?? null
  if (scoreView.value === 'stage') return a.stageSnapshot?.score ?? a.matchSnapshot?.score ?? a.match?.score ?? null
  return a.matchSnapshot?.score ?? a.stageSnapshot?.score ?? a.match?.score ?? null
}

const matchDist = computed(() => {
  const buckets = { sink: { label: '低匹配 0-59', count: 0, color: 'var(--red)' }, mid: { label: '一般 60-79', count: 0, color: 'var(--accent2)' }, hi: { label: '高匹配 80+', count: 0, color: 'var(--green)' } }
  store.applications.forEach(a => {
    const s = scoreOf(a)
    if (s == null) return
    if (s >= 80) buckets.hi.count++
    else if (s >= 60) buckets.mid.count++
    else buckets.sink.count++
  })
  return Object.values(buckets)
})
const matchTotal = computed(() => matchDist.value.reduce((s, b) => s + b.count, 0) || 1)

const avgOfView = view => {
  const pick = a => view === 'latest'
    ? (a.match?.score ?? a.stageSnapshot?.score ?? a.matchSnapshot?.score)
    : view === 'stage'
      ? (a.stageSnapshot?.score ?? a.matchSnapshot?.score ?? a.match?.score)
      : (a.matchSnapshot?.score ?? a.stageSnapshot?.score ?? a.match?.score)
  const vals = store.applications.map(pick).filter(s => s != null)
  return vals.length ? Math.round(vals.reduce((s, v) => s + v, 0) / vals.length) : 0
}
const avgSnapshot = computed(() => avgOfView('snapshot'))
const avgStage = computed(() => avgOfView('stage'))
const avgLatest = computed(() => avgOfView('latest'))
const avgCurrent = computed(() => ({ snapshot: avgSnapshot, stage: avgStage, latest: avgLatest }[scoreView.value].value))

const donutSegs = computed(() => matchDist.value.map((b, i) => {
  const len = (b.count / matchTotal.value) * 389.4
  return { ...b, len, offset: -(matchDist.value.slice(0, i).reduce((s, x) => s + ((x.count / matchTotal.value) * 389.4), 0)) }
}))

const avgSalary = computed(() => {
  const offers = store.offers.filter(o => o.status === 'accepted' || o.status === 'joined')
  return offers.length ? Math.round(offers.reduce((s, o) => s + o.salary, 0) / offers.length) : 0
})

const strategyGovernance = computed(() => {
  const count = status => store.strategyVersions.filter(v => v.status === status).length
  const jobs = store.recalcJobs.slice(0, 8)
  const canaryMatches = store.matches.filter(m => m.is_canary)
  const baselineMatches = store.matches.filter(m => !m.is_canary)
  const avg = rows => rows.length ? Math.round(rows.reduce((s, m) => s + m.score, 0) / rows.length) : 0
  return {
    pending: count('pending') + count('returned') + count('scheduled'),
    canary: count('canary'),
    full: count('full'),
    rolledBack: count('rolled_back'),
    canaryPairs: canaryMatches.length,
    canaryAvg: avg(canaryMatches),
    baselineAvg: avg(baselineMatches),
    totalJobs: store.recalcJobs.length,
    totalPairs: store.recalcJobs.reduce((s, j) => s + (Number(j.pair_count) || 0), 0),
    jobs
  }
})
const triggerLabel = {
  strategy_publish: '全量发布', strategy_canary: '灰度发布', strategy_promote: '灰度转正',
  strategy_rollback: '策略回滚', manual: '手动重算', startup: '启动迁移'
}

// ---------------- 面试预约风险报表（缺席/改期/重约统一口径） ----------------
// 数据源与招聘看板/预约沟通完全一致：applications.schedule_risk 由后端在任一预约状态变化（裁定/重约/改期/
// 危机挂起/恢复）同事务重算回写；责任归属以 appointment_adjudications 最终裁定为准（改判不丢历史）
const risk = computed(() => store.scheduleRiskSummary || {})
const RISK_LEVEL_LABEL = { high: '高风险', mid: '中风险', none: '无' }
const riskApps = computed(() =>
  store.applications
    .filter(a => a.scheduleRisk && a.scheduleRisk.appt_total > 0)
    .map(a => ({ a, r: a.scheduleRisk }))
    .sort((x, y) => ({ high: 2, mid: 1, none: 0 }[y.r.risk_level] - { high: 2, mid: 1, none: 0 }[x.r.risk_level]
      || y.r.noshow_total - x.r.noshow_total))
)
const riskInterviewers = computed(() => (risk.value.interviewer_responsibility || [])
  .filter(v => v.interviewer_noshow || v.both_noshow || v.candidate_noshow || v.reschedule_total)
  .slice(0, 8))
</script>

<template>
  <div class="reports" v-if="store.loaded">
    <div class="stat-grid">
      <div class="card stat"><span>🔻</span><b>{{ overallConv }}%</b><em>投递→录用总转化</em></div>
      <div class="card stat"><span>💵</span><b class="money">¥{{ avgSalary.toLocaleString() }}</b><em>接受 Offer 平均薪资</em></div>
      <div class="card stat"><span>🎉</span><b class="money">{{ joinedCount }}</b><em>已入职人数</em></div>
      <div class="card stat"><span>✅</span><b class="money">{{ offerAcceptedCount }}</b><em>已接受 Offer</em></div>
      <div class="card stat"><span>⏳</span><b>{{ pendingOfferCount }}</b><em>待回应 Offer</em></div>
    </div>

    <div class="row">
      <div class="card">
        <h3>🔻 招聘漏斗转化率
          <span class="mode-switch">
            口径：
            <button :class="{ on: funnelMode === 'current' }" @click="funnelMode = 'current'">当前阶段</button>
            <button :class="{ on: funnelMode === 'reached' }" @click="funnelMode = 'reached'">曾经到达</button>
          </span>
        </h3>
        <div class="funnel">
          <div v-for="(f, i) in funnel" :key="f.s">
            <div class="fl">
              <span class="flabel">{{ f.label }}</span>
              <b>{{ f.count }}</b>
              <span class="conv" v-if="f.conv != null">{{ f.conv }}%</span>
              <i class="arrow" v-if="i < funnel.length - 1">↓</i>
            </div>
          </div>
        </div>
        <div class="muted tip">「当前阶段」为实时快照口径；「曾经到达」按阶段事件去重统计，异常回退后候选人仍计入其到达过的阶段。</div>
      </div>

      <div class="card">
        <h3>💴 渠道成本与效率</h3>
        <div class="clist">
          <div v-for="c in channelCost" :key="c.name" class="crow">
            <span class="cname">{{ c.name }}</span>
            <span class="muted">获取 {{ c.count }} 人</span>
            <b class="cpc">¥{{ c.cpc }}</b>
            <span class="hired-tag" v-if="c.joined">入职{{ c.joined }}</span>
          </div>
        </div>
        <div class="muted tip">单位成本 = 渠道费用 ÷ 获取候选人数，入职数以 Offer 状态「已入职」为准，与 Offer 页同口径。</div>
      </div>

      <div class="card">
        <h3>🏢 部门招聘进度</h3>
        <div class="dlist">
          <div v-for="d in deptProgress" :key="d.dept" class="drow">
            <div class="dhead">
              <span>{{ d.dept }}</span>
              <span class="muted">录用 {{ d.hired }}/{{ d.total }} · 入职 {{ d.joined }}</span>
            </div>
            <div class="bar"><i :style="{ width: (d.hired / (d.total || 1)) * 100 + '%', background: 'var(--green)' }"></i></div>
          </div>
        </div>
      </div>
    </div>

    <div class="row2">
      <div class="card">
        <h3>🎯 候选人匹配度分布</h3>
        <div class="caliber">
          <button :class="{ on: scoreView === 'snapshot' }" @click="scoreView = 'snapshot'">🕘 投递时评分</button>
          <button :class="{ on: scoreView === 'stage' }" @click="scoreView = 'stage'">🧭 当前阶段</button>
          <button :class="{ on: scoreView === 'latest' }" @click="scoreView = 'latest'">🆕 最新结果</button>
        </div>
        <div class="donut-wrap">
          <svg viewBox="0 0 160 160" class="donut">
            <circle cx="80" cy="80" r="62" fill="none" stroke="#222a4a" stroke-width="26"/>
            <circle v-for="(s, i) in donutSegs" :key="s.label" cx="80" cy="80" r="62" fill="none"
              :stroke="s.color" stroke-width="26"
              :stroke-dasharray="s.len" :stroke-dashoffset="s.offset" transform="rotate(-90 80 80)"/>
          </svg>
          <div class="center"><b>{{ avgCurrent }}</b><em class="muted">{{ { snapshot: '投递时平均分', stage: '当前阶段平均分', latest: '最新平均分' }[scoreView] }}</em></div>
        </div>
        <div class="avg-cmp muted">
          投递 <b :class="avgSnapshot >= 75 ? 's-hi' : avgSnapshot >= 55 ? 's-mid' : 's-lo'">{{ avgSnapshot }}</b>
          → 当前阶段 <b :class="avgStage >= 75 ? 's-hi' : avgStage >= 55 ? 's-mid' : 's-lo'">{{ avgStage }}</b>
          → 最新 <b :class="avgLatest >= 75 ? 's-hi' : avgLatest >= 55 ? 's-mid' : 's-lo'">{{ avgLatest }}</b>
          <span v-if="avgLatest - avgSnapshot" :class="avgLatest - avgSnapshot > 0 ? 'up' : 'down'">
            （{{ avgLatest - avgSnapshot > 0 ? '+' : '' }}{{ avgLatest - avgSnapshot }}）
          </span>
        </div>
        <div class="leg">
          <div v-for="s in matchDist" :key="s.label"><span class="sw" :style="{background:s.color}"></span>{{ s.label }}<b>{{ s.count }}人</b></div>
        </div>
        <div class="muted tip">三种口径与招聘流程看板完全一致：投递快照永不改变、当前阶段快照随推进/回退刷新、最新结果仅由显式重算更新。</div>
      </div>

      <div class="card">
        <h3>📊 招聘效率概况</h3>
        <div class="kpis">
          <div><em class="muted">简历投递 → 筛选</em><b>{{ funnel[1]?.conv ?? 0 }}%</b></div>
          <div><em class="muted">筛选 → 面试</em><b>{{ funnel[2]?.conv ?? 0 }}%</b></div>
          <div><em class="muted">面试 → Offer（需通过结论）</em><b>{{ funnel[3]?.conv ?? 0 }}%</b></div>
          <div><em class="muted">Offer → 录用接受</em><b>{{ funnel[4]?.conv ?? 0 }}%</b></div>
          <div class="wide"><em class="muted">录用 → 入职（接受 Offer 后确认）</em><b>{{ offerJoinRate }}%</b></div>
        </div>
        <div class="muted tip">建议：若「筛选→面试」转化低，可优化职位JD与筛选标准；「Offer→录用」低则需复核薪酬竞争力与流程效率；「录用→入职」低需关注放鸽子与入职跟进。</div>
      </div>
    </div>

    <div class="card risk-card">
      <h3>⚠️ 面试预约风险报表（缺席 · 改期 · 重约）
        <span class="muted" style="font-size:11px;font-weight:400;margin-left:8px">责任裁定以最终裁定为准，改判/重约/危机恢复同事务回写，与招聘看板口径一致</span>
      </h3>
      <div class="risk-kpis">
        <div><em>高风险应聘</em><b class="r-high">{{ risk.high_risk_apps || 0 }}</b></div>
        <div><em>中风险应聘</em><b class="r-mid">{{ risk.mid_risk_apps || 0 }}</b></div>
        <div><em>候选人责任缺席</em><b class="r-high">{{ risk.candidate_noshow || 0 }}</b></div>
        <div><em>面试官责任缺席</em><b class="r-mid">{{ risk.interviewer_noshow || 0 }}</b></div>
        <div><em>双方缺席</em><b>{{ risk.both_noshow || 0 }}</b></div>
        <div><em>累计改期</em><b>{{ risk.reschedule_total || 0 }}</b></div>
        <div><em>待重约（缺席未闭环）</em><b class="r-mid">{{ risk.rebook_open || 0 }}</b></div>
        <div><em>危机挂起</em><b class="r-crisis">{{ risk.crisis_suspended || 0 }}</b></div>
        <div><em>责任到场率</em><b>{{ risk.attend_rate == null ? '—' : risk.attend_rate + '%' }}</b></div>
      </div>

      <div class="risk-cols">
        <div class="risk-table">
          <h4>在途应聘风险清单</h4>
          <div class="rt-row rt-head rt-cols-app"><span>候选人 · 职位</span><span>风险</span><span>责任缺席（候/官/双）</span><span>改期</span><span>待重约</span></div>
          <div class="rt-row rt-cols-app" v-for="({ a, r }) in riskApps.slice(0, 12)" :key="a.id">
            <span class="rt-name">{{ a.candidate }}<em class="muted"> · {{ a.position }}</em></span>
            <span class="rt-level" :class="r.risk_level">{{ RISK_LEVEL_LABEL[r.risk_level] }}</span>
            <span class="rt-nums"><i class="r-high">{{ r.candidate_noshow }}</i> / <i class="r-mid">{{ r.interviewer_noshow }}</i> / <i>{{ r.both_noshow }}</i></span>
            <span>{{ r.reschedule_total }}</span>
            <span :class="{ 'r-mid': r.rebook_open }">{{ r.rebook_open }}</span>
          </div>
          <div class="muted empty-mini" v-if="!riskApps.length">暂无已安排预约的应聘。</div>
        </div>
        <div class="risk-table">
          <h4>面试官责任画像（缺席 / 改期）</h4>
          <div class="rt-row rt-head rt-cols-iv"><span>面试官</span><span>面试官缺席</span><span>双方缺席</span><span>候选人缺席</span><span>改期</span></div>
          <div class="rt-row rt-cols-iv" v-for="v in riskInterviewers" :key="v.id">
            <span class="rt-name">{{ v.name }}</span>
            <span :class="{ 'r-mid': v.interviewer_noshow }">{{ v.interviewer_noshow || 0 }}</span>
            <span :class="{ 'r-mid': v.both_noshow }">{{ v.both_noshow || 0 }}</span>
            <span :class="{ 'r-high': v.candidate_noshow }">{{ v.candidate_noshow || 0 }}</span>
            <span>{{ v.reschedule_total || 0 }}</span>
          </div>
          <div class="muted empty-mini" v-if="!riskInterviewers.length">暂无预约记录。</div>
        </div>
      </div>
      <div class="muted tip">
        候选人/双方责任缺席未完成重约（或候选人发起的改期仍在协商）时，招聘阶段「面试 → Offer」推进被服务端硬拦截；面试官单方责任缺席不阻塞候选人流程但计入责任画像。
        责任到场率 = 已完成预约 ÷（完成 + 责任已落定的缺席），系统初判待裁定的记录不计入分母。
      </div>
    </div>

    <div class="card governance-card">
      <h3>🧪 策略版本治理与灰度效果</h3>
      <div class="gov-kpis">
        <div><em>待审/待生效</em><b>{{ strategyGovernance.pending }}</b></div>
        <div><em>灰度中版本</em><b>{{ strategyGovernance.canary }}</b></div>
        <div><em>全量版本</em><b>{{ strategyGovernance.full }}</b></div>
        <div><em>累计回滚</em><b>{{ strategyGovernance.rolledBack }}</b></div>
        <div><em>灰度命中匹配</em><b>{{ strategyGovernance.canaryPairs }}</b></div>
        <div><em>灰度/基线均分</em><b>{{ strategyGovernance.canaryAvg }} / {{ strategyGovernance.baselineAvg }}</b></div>
      </div>
      <div class="job-table">
        <div class="job-row job-head"><span>批次</span><span>触发</span><span>版本</span><span>灰度/基线</span><span>总分</span><span>状态</span></div>
        <div class="job-row" v-for="j in strategyGovernance.jobs" :key="j.id">
          <span>#{{ j.id }}</span>
          <span>{{ triggerLabel[j.trigger_type] || j.trigger_type }}</span>
          <span>{{ j.strategy_id ? `v${j.strategy_id}` : 'v0' }}</span>
          <span>{{ j.canary_count || 0 }} / {{ j.baseline_count || j.pair_count || 0 }}</span>
          <span>{{ j.pair_count }}</span>
          <span>{{ j.status }}</span>
        </div>
      </div>
      <div class="muted tip">报表只统计最新匹配与不可变重算批次；投递时评分和阶段评分快照仍按审批/投递当时版本统计，策略全量或回滚均不重写历史决策证据。</div>
    </div>
  </div>
</template>

<style scoped>
.reports { display: flex; flex-direction: column; gap: 16px; }
.stat-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(170px, 1fr)); gap: 14px; }
.stat { display: flex; flex-direction: column; gap: 4px; }
.stat span { font-size: 24px; }
.stat b { font-size: 24px; }
.stat em { font-style: normal; color: var(--muted); font-size: 13px; }
.row { display: grid; grid-template-columns: 1.2fr 1fr 1fr; gap: 16px; }
.row2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
@media (max-width: 1000px) { .row, .row2 { grid-template-columns: 1fr; } }
.mode-switch { font-size: 11px; font-weight: 400; margin-left: 8px; }
.mode-switch button { padding: 2px 8px; font-size: 11px; opacity: .75; }
.mode-switch button.on { opacity: 1; border-color: var(--accent); background: rgba(91,140,255,.15); color: var(--accent); }
.funnel { display: flex; flex-direction: column; gap: 2px; }
.fl { display: flex; align-items: center; gap: 12px; padding: 10px 12px; background: var(--panel2); border-radius: 8px; }
.flabel { width: 90px; }
.fl b { font-size: 20px; }
.conv { margin-left: auto; background: rgba(87,214,160,.16); color: var(--green); padding: 2px 8px; border-radius: 10px; font-size: 12px; }
.arrow { color: var(--muted); text-align: center; display: block; padding: 2px 0 2px 100px; }
.clist, .dlist { display: flex; flex-direction: column; gap: 10px; }
.crow { display: flex; align-items: center; gap: 12px; padding: 9px 10px; background: var(--panel2); border-radius: 8px; font-size: 13px; }
.cname { width: 90px; font-weight: 600; }
.cpc { margin-left: auto; font-size: 16px; color: var(--accent2); }
.hired-tag { font-size: 11px; background: rgba(87,214,160,.15); color: var(--green); padding: 2px 8px; border-radius: 10px; }
.tip { margin-top: 10px; line-height: 1.6; }
.drow { display: flex; flex-direction: column; gap: 6px; }
.dhead { display: flex; justify-content: space-between; font-size: 13px; }
.bar { height: 9px; background: var(--panel2); border-radius: 5px; overflow: hidden; }
.bar i { display: block; height: 100%; transition: .4s; }
.donut-wrap { position: relative; width: 160px; margin: 6px auto; }
.donut { width: 160px; height: 160px; }
.center { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; }
.center b { font-size: 26px; }
.caliber { display: flex; gap: 6px; margin-bottom: 10px; flex-wrap: wrap; }
.caliber button { padding: 5px 10px; font-size: 12px; opacity: .75; }
.caliber button.on { opacity: 1; border-color: var(--accent); background: rgba(91,140,255,.15); color: var(--accent); }
.avg-cmp { text-align: center; font-size: 12px; margin-top: 4px; }
.avg-cmp b { padding: 1px 7px; border-radius: 8px; margin: 0 2px; }
.avg-cmp .up { color: var(--green); }
.avg-cmp .down { color: var(--red); }
.leg { display: flex; flex-direction: column; gap: 6px; margin-top: 10px; }
.leg div { display: flex; align-items: center; gap: 6px; font-size: 13px; }
.leg b { margin-left: auto; }
.sw { width: 10px; height: 10px; border-radius: 3px; }
.kpis { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.kpis div { background: var(--panel2); border-radius: 10px; padding: 14px; text-align: center; }
.kpis div.wide { grid-column: 1 / -1; }
.kpis em { display: block; font-style: normal; font-size: 12px; }
.kpis b { font-size: 24px; color: var(--accent); }
.governance-card { margin-top: 0; }
.gov-kpis { display: grid; grid-template-columns: repeat(auto-fit, minmax(130px, 1fr)); gap: 10px; margin-bottom: 14px; }
.gov-kpis div { background: var(--panel2); border-radius: 10px; padding: 12px; text-align: center; }
.gov-kpis em { display: block; font-style: normal; font-size: 12px; color: var(--muted); }
.gov-kpis b { display: block; font-size: 21px; color: var(--purple); margin-top: 4px; }
.job-table { border: 1px solid var(--border); border-radius: 10px; overflow: hidden; }
.job-row { display: grid; grid-template-columns: .6fr 1fr .6fr 1fr .6fr .8fr; gap: 8px; padding: 8px 10px; font-size: 12px; }
.job-row:nth-child(odd):not(.job-head) { background: var(--panel2); }
.job-head { background: rgba(91,140,255,.1); font-weight: 700; color: var(--accent); }

/* 面试预约风险报表 */
.risk-kpis { display: grid; grid-template-columns: repeat(auto-fit, minmax(120px, 1fr)); gap: 10px; margin-bottom: 14px; }
.risk-kpis div { background: var(--panel2); border-radius: 10px; padding: 11px; text-align: center; }
.risk-kpis em { display: block; font-style: normal; font-size: 11.5px; color: var(--muted); }
.risk-kpis b { display: block; font-size: 21px; margin-top: 4px; }
.risk-kpis b.r-high { color: var(--red); }
.risk-kpis b.r-mid { color: var(--accent2); }
.risk-kpis b.r-crisis { color: var(--purple); }
.risk-cols { display: grid; grid-template-columns: 1.35fr 1fr; gap: 14px; }
@media (max-width: 1000px) { .risk-cols { grid-template-columns: 1fr; } }
.risk-table h4 { font-size: 13px; margin-bottom: 8px; }
.rt-row { display: grid; gap: 8px; align-items: center; padding: 7px 9px; font-size: 12px; border-radius: 8px; }
.rt-cols-app { grid-template-columns: 1.6fr .7fr 1.3fr .6fr .7fr; }
.rt-cols-iv { grid-template-columns: 1.4fr .7fr 1fr 1fr .6fr; }
.rt-row:nth-child(odd) { background: var(--panel2); }
.rt-head { font-weight: 700; color: var(--accent); background: rgba(91,140,255,.08) !important; }
.rt-name em { font-style: normal; opacity: .75; }
.rt-level { border-radius: 9px; padding: 1px 8px; text-align: center; border: 1px solid var(--border); color: var(--muted); font-size: 11px; }
.rt-level.high { color: var(--red); border-color: rgba(255,107,122,.45); background: rgba(255,107,122,.08); }
.rt-level.mid { color: var(--accent2); border-color: rgba(255,209,102,.45); background: rgba(255,209,102,.08); }
.rt-level.none { color: var(--green); border-color: rgba(87,214,160,.35); }
.rt-nums i { font-style: normal; }
.r-high { color: var(--red); font-weight: 700; }
.r-mid { color: var(--accent2); font-weight: 700; }
.empty-mini { padding: 8px 4px; font-size: 12px; text-align: center; }
</style>
