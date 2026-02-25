/**
 * ============================================================
 * SERVER — server.js
 *
 * Express entry point. Wires middleware, routes, and static
 * file serving together. Kept minimal by design.
 * ============================================================
 */

'use strict';

const express = require('express');
const path    = require('path');

const decisionRoutes = require('./src/routes/decision');

const app  = express();
const PORT = process.env.PORT || 3000;

/* ── MIDDLEWARE ─────────────────────────────────────────────── */
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

/* Request logger (dev only) */
app.use((req, _res, next) => {
  if (process.env.NODE_ENV !== 'production') {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.path}`);
  }
  next();
});

/* ── STATIC FILES (frontend) ────────────────────────────────── */
app.use(express.static(path.join(__dirname, 'public')));

/* ── API ROUTES ─────────────────────────────────────────────── */
app.use('/api/decision', decisionRoutes);

/* ── HEALTH CHECK ───────────────────────────────────────────── */
app.get('/api/health', (_req, res) => {
  res.json({ ok: true, service: 'Decision Helper API', ts: new Date().toISOString() });
});

/* ── SPA FALLBACK ───────────────────────────────────────────── */
app.get('*', (_req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

/* ── ERROR HANDLER ──────────────────────────────────────────── */
app.use((err, _req, res, _next) => {
  console.error('[Unhandled]', err.message);
  res.status(500).json({ ok: false, error: 'Internal server error.' });
});

app.listen(PORT, () => {
  console.log(`\n🧠 Decision Helper running → http://localhost:${PORT}\n`);
});

module.exports = app;
/**
 * ============================================================
 * DECISION ENGINE — src/engine/scoringEngine.js
 *
 * Layered scoring architecture:
 *   1. Raw score accumulation (weighted answers)
 *   2. Category normalization (0–100 per dimension)
 *   3. Risk analysis (flags high-risk patterns)
 *   4. Composite recommendation generation
 *
 * AI-ready: each layer can be replaced by an ML model call.
 * ============================================================
 */

'use strict';

/* ── WEIGHT REGISTRY ─────────────────────────────────────────
   Each question has an `impact` multiplier (0.5–3.0).
   Higher = this answer swings the decision more heavily.
   Categories map to outcome dimensions.
   ─────────────────────────────────────────────────────────── */
const CATEGORY_WEIGHTS = {
  feasibility:  1.5,   // Can we actually do this?
  risk:         2.0,   // How bad if it goes wrong?
  impact:       1.8,   // How good if it succeeds?
  resources:    1.3,   // Do we have what we need?
  urgency:      1.0,   // How time-sensitive?
};

/* ── RISK THRESHOLD CONFIG ───────────────────────────────────
   Drives the risk analysis layer. Configurable per deployment.
   ─────────────────────────────────────────────────────────── */
const RISK_CONFIG = {
  highRiskThreshold:   35,   // raw risk score → flag as HIGH
  lowFeasibilityWarn:  40,   // below this → feasibility warning
  criticalComboFlags: [      // combinations that auto-flag
    ['risk', 'resources'],   // high risk + low resources = critical
    ['risk', 'feasibility'], // high risk + low feasibility = critical
  ],
};

/* ── SCORING ENGINE CLASS ─────────────────────────────────── */
class ScoringEngine {
  /**
   * Process a completed answer set and produce a scored result.
   *
   * @param {Array}  answers  - [{ questionId, value, category, weight }]
   * @param {Object} metadata - { decisionTitle, sessionId }
   * @returns {Object} Full scored result object
   */
  static score(answers, metadata = {}) {
    // Layer 1: accumulate raw scores per category
    const rawScores = this._accumulateRaw(answers);

    // Layer 2: normalize to 0–100 per category
    const normalized = this._normalize(rawScores);

    // Layer 3: risk analysis
    const riskReport = this._analyzeRisk(normalized, answers);

    // Layer 4: generate recommendation
    const recommendation = this._recommend(normalized, riskReport);

    // Layer 5: build confidence score (meta-layer)
    const confidence = this._calculateConfidence(normalized, riskReport, answers.length);

    return {
      sessionId:      metadata.sessionId,
      decisionTitle:  metadata.decisionTitle || 'Unnamed Decision',
      scores:         normalized,
      rawScores,
      riskReport,
      recommendation,
      confidence,
      totalAnswers:   answers.length,
      scoredAt:       new Date().toISOString(),
    };
  }

  /* ── LAYER 1: RAW ACCUMULATION ──────────────────────────── */
  static _accumulateRaw(answers) {
    const sums   = {};
    const counts = {};

    for (const { category, value, weight = 1 } of answers) {
      if (!category) continue;
      const cat = category.toLowerCase();
      sums[cat]   = (sums[cat]   || 0) + (value * weight);
      counts[cat] = (counts[cat] || 0) + weight;
    }

    // Weighted average per category → raw score 0–10
    const raw = {};
    for (const cat of Object.keys(sums)) {
      raw[cat] = counts[cat] > 0 ? sums[cat] / counts[cat] : 0;
    }
    return raw;
  }

  /* ── LAYER 2: NORMALIZATION ──────────────────────────────── */
  static _normalize(rawScores) {
    const normalized = {};
    for (const [cat, raw] of Object.entries(rawScores)) {
      // Raw is 0–10; apply category weight; clamp to 0–100
      const catWeight = CATEGORY_WEIGHTS[cat] || 1.0;
      const scaled    = (raw / 10) * 100 * Math.min(catWeight / 2 + 0.5, 1.5);
      normalized[cat] = Math.min(100, Math.max(0, Math.round(scaled)));
    }
    return normalized;
  }

  /* ── LAYER 3: RISK ANALYSIS ──────────────────────────────── */
  static _analyzeRisk(normalized, answers) {
    const flags    = [];
    const warnings = [];
    let   severity = 'LOW'; // LOW | MEDIUM | HIGH | CRITICAL

    // Check individual thresholds
    const riskScore       = normalized.risk        || 0;
    const feasibility     = normalized.feasibility || 100;
    const resourceScore   = normalized.resources   || 100;

    // Invert risk: high raw score on risk questions = high risk
    const effectiveRisk = 100 - riskScore;

    if (effectiveRisk >= RISK_CONFIG.highRiskThreshold) {
      flags.push({ code: 'HIGH_RISK', message: 'Risk exposure is significantly elevated.' });
      severity = 'HIGH';
    }
    if (feasibility < RISK_CONFIG.lowFeasibilityWarn) {
      warnings.push({ code: 'LOW_FEASIBILITY', message: 'Feasibility score is below safe threshold.' });
      if (severity !== 'CRITICAL') severity = 'MEDIUM';
    }
    if (resourceScore < 40) {
      warnings.push({ code: 'RESOURCE_CONSTRAINT', message: 'Insufficient resources detected.' });
      if (severity !== 'CRITICAL') severity = 'MEDIUM';
    }

    // Check critical combinations
    for (const [a, b] of RISK_CONFIG.criticalComboFlags) {
      const scoreA = a === 'risk' ? effectiveRisk : (normalized[a] || 100);
      const scoreB = b === 'risk' ? effectiveRisk : (normalized[b] || 100);
      if (scoreA >= 50 && scoreB < 40) {
        flags.push({
          code: 'CRITICAL_COMBO',
          message: `Critical flag: high ${a} exposure combined with low ${b}.`,
        });
        severity = 'CRITICAL';
      }
    }

    return {
      severity,
      effectiveRiskScore: effectiveRisk,
      flags,
      warnings,
      safe: severity === 'LOW' || severity === 'MEDIUM',
    };
  }

  /* ── LAYER 4: RECOMMENDATION ENGINE ─────────────────────── */
  static _recommend(normalized, riskReport) {
    const impact      = normalized.impact      || 50;
    const feasibility = normalized.feasibility || 50;
    const urgency     = normalized.urgency     || 50;
    const resources   = normalized.resources   || 50;

    // Composite opportunity score (higher = more attractive)
    const opportunity = Math.round(
      (impact * 0.4) + (feasibility * 0.3) + (resources * 0.2) + (urgency * 0.1)
    );

    // Adjust downward for risk severity
    const riskPenalty = { LOW: 0, MEDIUM: 8, HIGH: 20, CRITICAL: 35 };
    const adjusted    = Math.max(0, opportunity - (riskPenalty[riskReport.severity] || 0));

    let decision, rationale, action;

    if (riskReport.severity === 'CRITICAL') {
      decision  = 'DO NOT PROCEED';
      rationale = 'Critical risk factors outweigh the potential benefits. Fundamental conditions must change before this decision can be safely made.';
      action    = 'Pause and reassess. Address flagged critical risks first.';
    } else if (adjusted >= 75) {
      decision  = 'PROCEED';
      rationale = 'Strong opportunity score with manageable risk profile. Conditions are favorable.';
      action    = 'Move forward with standard monitoring and contingency planning.';
    } else if (adjusted >= 55) {
      decision  = 'PROCEED WITH CAUTION';
      rationale = 'Moderate opportunity with elevated risk or resource constraints. Viable but requires mitigation.';
      action    = 'Define clear success criteria and set trigger points for reassessment.';
    } else if (adjusted >= 35) {
      decision  = 'DEFER';
      rationale = 'Insufficient conditions for a confident decision. Key dimensions score below threshold.';
      action    = 'Improve feasibility or resources before revisiting. Set a review date.';
    } else {
      decision  = 'DO NOT PROCEED';
      rationale = 'Opportunity score is low and risk factors are significant. This decision is not justified under current conditions.';
      action    = 'Abandon or fundamentally restructure this option.';
    }

    return {
      decision,
      rationale,
      action,
      opportunityScore: opportunity,
      adjustedScore:    adjusted,
    };
  }

  /* ── LAYER 5: CONFIDENCE SCORING ────────────────────────── */
  static _calculateConfidence(normalized, riskReport, answerCount) {
    // More answers = more confidence (cap at 10 for full confidence)
    const answerCoverage = Math.min(answerCount / 10, 1);

    // Fewer risk flags = more confidence
    const flagPenalty = (riskReport.flags.length * 0.1) + (riskReport.warnings.length * 0.05);

    // Score spread: narrow spread = lower confidence (one category dominates)
    const scores = Object.values(normalized);
    const avg    = scores.reduce((s, v) => s + v, 0) / (scores.length || 1);
    const spread = scores.reduce((s, v) => s + Math.abs(v - avg), 0) / (scores.length || 1);
    const spreadFactor = 1 - Math.min(spread / 60, 0.3);

    const confidence = Math.round(
      Math.max(0, Math.min(100,
        (answerCoverage * 70) + (spreadFactor * 30) - (flagPenalty * 30)
      ))
    );

    const label = confidence >= 75 ? 'HIGH'
                : confidence >= 50 ? 'MODERATE'
                : 'LOW';

    return { score: confidence, label };
  }
}

module.exports = ScoringEngine;
/**
 * ============================================================
 * QUESTION BANK — src/engine/questionBank.js
 *
 * Defines the decision framework's question set.
 * Designed for extensibility: add new questions or categories
 * without touching any other layer.
 *
 * Question schema:
 * {
 *   id:        string   unique identifier
 *   text:      string   question shown to user
 *   category:  string   scoring dimension
 *   weight:    number   1.0 = normal, 2.0 = double impact
 *   type:      string   'scale' | 'choice' | 'boolean'
 *   options:   Array    choices for type:'choice'
 *   min/max:   number   for type:'scale'
 *   hint:      string   helper text shown in UI
 * }
 * ============================================================
 */

'use strict';

const QUESTIONS = [
  /* ── FEASIBILITY ─────────────────────────────────────────── */
  {
    id:       'q1',
    text:     'How clearly defined is the goal or outcome you want to achieve?',
    category: 'feasibility',
    weight:   1.5,
    type:     'scale',
    min:      1,
    max:      10,
    hint:     '1 = vague idea, 10 = crystal clear with measurable criteria',
  },
  {
    id:       'q2',
    text:     'How confident are you in your knowledge of the domain or field this decision falls within?',
    category: 'feasibility',
    weight:   1.0,
    type:     'scale',
    min:      1,
    max:      10,
    hint:     '1 = completely new territory, 10 = deep expertise',
  },

  /* ── RISK ─────────────────────────────────────────────────── */
  {
    id:       'q3',
    text:     'If this decision goes wrong, how severe would the consequences be?',
    category: 'risk',
    weight:   2.0,
    type:     'choice',
    hint:     'Think about financial, reputational, and personal impact.',
    options: [
      { label: 'Negligible — easily reversible',       value: 9 },
      { label: 'Minor — some inconvenience',           value: 7 },
      { label: 'Moderate — significant setback',       value: 5 },
      { label: 'Severe — major long-term consequence', value: 3 },
      { label: 'Catastrophic — irreversible damage',   value: 1 },
    ],
  },
  {
    id:       'q4',
    text:     'How reversible is this decision once made?',
    category: 'risk',
    weight:   1.5,
    type:     'choice',
    hint:     'Can you undo it if needed?',
    options: [
      { label: 'Fully reversible at any time',    value: 10 },
      { label: 'Reversible with some cost',       value: 7  },
      { label: 'Partially reversible',            value: 5  },
      { label: 'Very difficult to reverse',       value: 3  },
      { label: 'Completely irreversible',         value: 1  },
    ],
  },

  /* ── IMPACT ──────────────────────────────────────────────── */
  {
    id:       'q5',
    text:     'What is the potential positive impact if this decision succeeds?',
    category: 'impact',
    weight:   1.8,
    type:     'choice',
    hint:     'Think broadly — financial, personal, organizational, societal.',
    options: [
      { label: 'Transformational — changes everything',  value: 10 },
      { label: 'Major — significant lasting benefit',    value: 8  },
      { label: 'Moderate — meaningful improvement',      value: 6  },
      { label: 'Minor — small incremental gain',         value: 4  },
      { label: 'Negligible — barely noticeable',         value: 2  },
    ],
  },
  {
    id:       'q6',
    text:     'How aligned is this decision with your core objectives or values?',
    category: 'impact',
    weight:   1.2,
    type:     'scale',
    min:      1,
    max:      10,
    hint:     '1 = completely misaligned, 10 = perfectly aligned',
  },

  /* ── RESOURCES ───────────────────────────────────────────── */
  {
    id:       'q7',
    text:     'Do you have sufficient financial resources to execute this decision?',
    category: 'resources',
    weight:   1.3,
    type:     'choice',
    hint:     'Consider both immediate and ongoing costs.',
    options: [
      { label: 'More than enough — comfortable buffer',  value: 10 },
      { label: 'Sufficient — will cover needs',          value: 7  },
      { label: 'Tight — manageable but stressful',       value: 5  },
      { label: 'Insufficient — will require borrowing',  value: 3  },
      { label: 'None — cannot fund this',                value: 1  },
    ],
  },
  {
    id:       'q8',
    text:     'Do you have the right people or team with the skills needed?',
    category: 'resources',
    weight:   1.1,
    type:     'scale',
    min:      1,
    max:      10,
    hint:     '1 = major skills gap, 10 = perfect team in place',
  },

  /* ── URGENCY ─────────────────────────────────────────────── */
  {
    id:       'q9',
    text:     'How time-sensitive is this decision?',
    category: 'urgency',
    weight:   1.0,
    type:     'choice',
    hint:     'What happens if you wait 3–6 months?',
    options: [
      { label: 'Critical — opportunity closes within days', value: 10 },
      { label: 'Urgent — must decide within weeks',         value: 8  },
      { label: 'Moderate — months available',               value: 6  },
      { label: 'Low — no real deadline',                    value: 4  },
      { label: 'None — timing doesn\'t matter',             value: 2  },
    ],
  },
  {
    id:       'q10',
    text:     'How much preparation or research have you done for this decision?',
    category: 'feasibility',
    weight:   1.2,
    type:     'scale',
    min:      1,
    max:      10,
    hint:     '1 = none at all, 10 = exhaustive research completed',
  },
];

/**
 * Retrieve all questions (ordered for the multi-step form)
 */
function getAllQuestions() {
  return QUESTIONS;
}

/**
 * Retrieve a single question by ID
 */
function getQuestionById(id) {
  return QUESTIONS.find(q => q.id === id) || null;
}

/**
 * Get the total number of questions (used for progress calculation)
 */
function getQuestionCount() {
  return QUESTIONS.length;
}

module.exports = { getAllQuestions, getQuestionById, getQuestionCount };
/**
 * ============================================================
 * SESSION MANAGER — src/engine/sessionManager.js
 *
 * In-memory session store for tracking decision progress.
 * AI-ready: swap the in-memory store for Redis/Postgres
 * by replacing the `store` implementation below.
 *
 * Session lifecycle:
 *   start → [answer × N] → result → (expires)
 * ============================================================
 */

'use strict';

/* ── SESSION STORE ───────────────────────────────────────────
   Production: replace with Redis client or ORM repository.
   The interface contract is: get(id), set(id, data), delete(id)
   ─────────────────────────────────────────────────────────── */
const store = new Map();

/* Auto-expire sessions after 2 hours (in ms) */
const SESSION_TTL_MS = 2 * 60 * 60 * 1000;

class SessionManager {
  /**
   * Create a new decision session.
   * @param {string} sessionId  - UUID from caller
   * @param {string} title      - User-provided decision title
   * @param {number} totalQ     - Total question count
   * @returns {Object} New session object
   */
  static create(sessionId, title, totalQ) {
    const session = {
      id:          sessionId,
      title,
      totalQ,
      answers:     [],           // { questionId, value, category, weight }
      status:      'IN_PROGRESS',// IN_PROGRESS | COMPLETE | EXPIRED
      startedAt:   new Date().toISOString(),
      updatedAt:   new Date().toISOString(),
      expiresAt:   new Date(Date.now() + SESSION_TTL_MS).toISOString(),
      result:      null,
    };

    store.set(sessionId, session);
    this._scheduleExpiry(sessionId);
    return session;
  }

  /**
   * Retrieve a session by ID.
   * @throws {Error} if session not found or expired
   */
  static get(sessionId) {
    const session = store.get(sessionId);
    if (!session) throw new Error(`Session not found: ${sessionId}`);
    if (session.status === 'EXPIRED') throw new Error('Session has expired.');
    return session;
  }

  /**
   * Append an answer to an existing session.
   * Replaces an existing answer for the same question if present.
   */
  static addAnswer(sessionId, answer) {
    const session = this.get(sessionId);

    // Idempotent: replace if question already answered
    const existing = session.answers.findIndex(a => a.questionId === answer.questionId);
    if (existing >= 0) {
      session.answers[existing] = answer;
    } else {
      session.answers.push(answer);
    }

    session.updatedAt = new Date().toISOString();

    // Mark complete when all questions answered
    if (session.answers.length >= session.totalQ) {
      session.status = 'COMPLETE';
    }

    store.set(sessionId, session);
    return session;
  }

  /**
   * Attach the final scored result to the session.
   */
  static attachResult(sessionId, result) {
    const session    = this.get(sessionId);
    session.result   = result;
    session.status   = 'COMPLETE';
    session.updatedAt = new Date().toISOString();
    store.set(sessionId, session);
    return session;
  }

  /**
   * Check whether all questions have been answered.
   */
  static isComplete(sessionId) {
    const session = this.get(sessionId);
    return session.answers.length >= session.totalQ;
  }

  /**
   * Return the count of answered vs total questions.
   */
  static progress(sessionId) {
    const session = this.get(sessionId);
    return { answered: session.answers.length, total: session.totalQ };
  }

  /* ── PRIVATE ─────────────────────────────────────────────── */
  static _scheduleExpiry(sessionId) {
    setTimeout(() => {
      const session = store.get(sessionId);
      if (session && session.status === 'IN_PROGRESS') {
        session.status = 'EXPIRED';
        store.set(sessionId, session);
      }
    }, SESSION_TTL_MS);
  }
}

module.exports = SessionManager;
/**
 * ============================================================
 * VALIDATORS — src/middleware/validators.js
 *
 * Input validation middleware. Keeps routes thin by moving
 * all validation logic here. AI-ready: swap for Zod/Joi.
 * ============================================================
 */

'use strict';

const { getQuestionById } = require('../engine/questionBank');

/* Generic error responder */
const fail = (res, msg, code = 400) => res.status(code).json({ ok: false, error: msg });

/* ── START ENDPOINT VALIDATOR ─────────────────────────────── */
function validateStart(req, res, next) {
  const { title } = req.body;
  if (!title || typeof title !== 'string' || title.trim().length < 2) {
    return fail(res, 'Provide a decision title (at least 2 characters).');
  }
  if (title.length > 120) {
    return fail(res, 'Title must be under 120 characters.');
  }
  next();
}

/* ── ANSWER ENDPOINT VALIDATOR ────────────────────────────── */
function validateAnswer(req, res, next) {
  const { sessionId, questionId, value } = req.body;

  if (!sessionId || typeof sessionId !== 'string') {
    return fail(res, 'sessionId is required.');
  }
  if (!questionId || typeof questionId !== 'string') {
    return fail(res, 'questionId is required.');
  }

  const question = getQuestionById(questionId);
  if (!question) {
    return fail(res, `Unknown questionId: ${questionId}`);
  }

  // Value must be numeric and within valid range
  const numeric = Number(value);
  if (isNaN(numeric)) {
    return fail(res, 'Answer value must be a number.');
  }
  if (numeric < 1 || numeric > 10) {
    return fail(res, 'Answer value must be between 1 and 10.');
  }

  // Attach validated question to request for route handler
  req.validatedQuestion = question;
  req.validatedValue    = numeric;
  next();
}

/* ── RESULT ENDPOINT VALIDATOR ────────────────────────────── */
function validateResult(req, res, next) {
  const { sessionId } = req.body;
  if (!sessionId || typeof sessionId !== 'string') {
    return fail(res, 'sessionId is required.');
  }
  next();
}

module.exports = { validateStart, validateAnswer, validateResult };
/**
 * ============================================================
 * DECISION ROUTES — src/routes/decision.js
 *
 * API contract:
 *
 *   POST /api/decision/start
 *     Body:    { title: string }
 *     Returns: { ok, sessionId, questions[], totalQuestions }
 *
 *   POST /api/decision/answer
 *     Body:    { sessionId, questionId, value }
 *     Returns: { ok, sessionId, progress, nextQuestion|null }
 *
 *   POST /api/decision/result
 *     Body:    { sessionId }
 *     Returns: { ok, result: ScoredResult }
 *
 * Routes are kept intentionally thin. All logic lives in the
 * engine layer — routes only orchestrate and respond.
 * ============================================================
 */

'use strict';

const express        = require('express');
const { v4: uuidv4 } = require('uuid');
const router         = express.Router();

const ScoringEngine   = require('../engine/scoringEngine');
const SessionManager  = require('../engine/sessionManager');
const { getAllQuestions, getQuestionCount } = require('../engine/questionBank');
const { validateStart, validateAnswer, validateResult } = require('../middleware/validators');

/* ── POST /api/decision/start ─────────────────────────────── */
router.post('/start', validateStart, (req, res) => {
  try {
    const sessionId = uuidv4();
    const questions = getAllQuestions();
    const totalQ    = getQuestionCount();

    SessionManager.create(sessionId, req.body.title.trim(), totalQ);

    res.json({
      ok:             true,
      sessionId,
      decisionTitle:  req.body.title.trim(),
      questions,          // Full question list — client drives the flow
      totalQuestions: totalQ,
      startedAt:      new Date().toISOString(),
    });
  } catch (err) {
    console.error('[/start]', err.message);
    res.status(500).json({ ok: false, error: 'Failed to start session.' });
  }
});

/* ── POST /api/decision/answer ────────────────────────────── */
router.post('/answer', validateAnswer, (req, res) => {
  try {
    const { sessionId } = req.body;
    const question      = req.validatedQuestion;
    const value         = req.validatedValue;

    const answer = {
      questionId: question.id,
      value,
      category:   question.category,
      weight:     question.weight,
    };

    const session    = SessionManager.addAnswer(sessionId, answer);
    const { answered, total } = SessionManager.progress(sessionId);

    // Work out which question is next
    const questions  = getAllQuestions();
    const answeredIds = new Set(session.answers.map(a => a.questionId));
    const nextQuestion = questions.find(q => !answeredIds.has(q.id)) || null;

    res.json({
      ok:           true,
      sessionId,
      progress:     { answered, total, percentComplete: Math.round((answered / total) * 100) },
      nextQuestion,
      isComplete:   answered >= total,
    });
  } catch (err) {
    console.error('[/answer]', err.message);
    const status = err.message.includes('not found') ? 404 : 500;
    res.status(status).json({ ok: false, error: err.message });
  }
});

/* ── POST /api/decision/result ────────────────────────────── */
router.post('/result', validateResult, (req, res) => {
  try {
    const { sessionId } = req.body;
    const session       = SessionManager.get(sessionId);

    // Return cached result if already scored
    if (session.result) {
      return res.json({ ok: true, result: session.result, cached: true });
    }

    if (session.answers.length === 0) {
      return res.status(400).json({ ok: false, error: 'No answers recorded yet.' });
    }

    // Score through the engine
    const result = ScoringEngine.score(session.answers, {
      sessionId,
      decisionTitle: session.title,
    });

    SessionManager.attachResult(sessionId, result);

    res.json({ ok: true, result, cached: false });
  } catch (err) {
    console.error('[/result]', err.message);
    const status = err.message.includes('not found') ? 404 : 500;
    res.status(status).json({ ok: false, error: err.message });
  }
});

/* ── GET /api/decision/questions ──────────────────────────── */
/* Helper endpoint — useful for AI integrations to inspect schema */
router.get('/questions', (_req, res) => {
  res.json({ ok: true, questions: getAllQuestions() });
});

module.exports = router;
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Decision Helper — Advanced Logic Engine</title>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link href="https://fonts.googleapis.com/css2?family=Syne:wght@400;700;800&family=JetBrains+Mono:wght@300;400;500&display=swap" rel="stylesheet" />
  <link rel="stylesheet" href="/css/app.css" />
</head>
<body>

  <!-- Grid overlay texture -->
  <div class="grid-overlay" aria-hidden="true"></div>

  <div class="app-shell">

    <!-- ── SCREEN 1: INTRO ─────────────────────────────── -->
    <section class="screen" id="screen-intro" data-active="true">
      <div class="intro-wrap">
        <div class="logo-block">
          <span class="logo-mark">◈</span>
          <div>
            <h1 class="logo-title">DECISION<br>HELPER</h1>
            <p class="logo-sub">Advanced Logic Engine v1.0</p>
          </div>
        </div>

        <div class="intro-body">
          <p class="intro-desc">
            A structured, weighted decision framework. Answer 10 calibrated questions.
            Receive a risk-analyzed recommendation backed by a multi-dimensional scoring engine.
          </p>

          <div class="input-group">
            <label class="field-label" for="decisionTitle">WHAT DECISION ARE YOU FACING?</label>
            <input
              type="text"
              id="decisionTitle"
              class="text-input"
              placeholder="e.g. 'Launch the new product line in Q3'"
              maxlength="120"
              autocomplete="off"
            />
            <span class="char-count" id="charCount">0 / 120</span>
          </div>

          <button class="btn-primary" id="btnStart" disabled>
            <span>BEGIN ANALYSIS</span>
            <span class="btn-arrow">→</span>
          </button>
        </div>

        <div class="system-info">
          <span>WEIGHTED SCORING</span>
          <span class="sep">·</span>
          <span>RISK ANALYSIS</span>
          <span class="sep">·</span>
          <span>5 DIMENSIONS</span>
        </div>
      </div>
    </section>

    <!-- ── SCREEN 2: QUESTIONS ─────────────────────────── -->
    <section class="screen" id="screen-questions" data-active="false">
      <header class="q-header">
        <div class="q-meta">
          <span class="session-label">SESSION ACTIVE</span>
          <span class="decision-title-display" id="qDecisionTitle"></span>
        </div>
        <div class="progress-wrap">
          <div class="progress-bar-track">
            <div class="progress-bar-fill" id="progressFill"></div>
          </div>
          <span class="progress-text" id="progressText">0 / 10</span>
        </div>
      </header>

      <main class="q-main">
        <!-- Category tag -->
        <div class="q-category-tag" id="qCategory"></div>

        <!-- Question number -->
        <div class="q-number" id="qNumber"></div>

        <!-- Question text -->
        <h2 class="q-text" id="qText"></h2>

        <!-- Hint -->
        <p class="q-hint" id="qHint"></p>

        <!-- Answer zone — dynamically populated by JS -->
        <div class="answer-zone" id="answerZone"></div>

        <!-- Navigation -->
        <div class="q-nav">
          <div class="q-weight-indicator" id="qWeightIndicator"></div>
          <button class="btn-primary" id="btnNext" disabled>
            <span>NEXT</span>
            <span class="btn-arrow">→</span>
          </button>
        </div>
      </main>
    </section>

    <!-- ── SCREEN 3: RESULT ─────────────────────────────── -->
    <section class="screen" id="screen-result" data-active="false">
      <div class="result-wrap">

        <!-- Header -->
        <div class="result-header">
          <span class="result-label">ANALYSIS COMPLETE</span>
          <h2 class="result-decision-name" id="rDecisionTitle"></h2>
        </div>

        <!-- Primary verdict -->
        <div class="verdict-block" id="verdictBlock">
          <div class="verdict-label">RECOMMENDATION</div>
          <div class="verdict-text" id="verdictText"></div>
          <div class="verdict-meta" id="verdictMeta"></div>
        </div>

        <!-- Score grid -->
        <div class="score-grid" id="scoreGrid"></div>

        <!-- Risk report -->
        <div class="risk-panel" id="riskPanel">
          <div class="risk-header">
            <span class="risk-title">RISK REPORT</span>
            <span class="risk-badge" id="riskBadge"></span>
          </div>
          <div class="risk-items" id="riskItems"></div>
        </div>

        <!-- Action block -->
        <div class="action-block">
          <div class="action-label">RECOMMENDED ACTION</div>
          <p class="action-text" id="actionText"></p>
        </div>

        <!-- Rationale -->
        <div class="rationale-block">
          <div class="rationale-label">SCORING RATIONALE</div>
          <p class="rationale-text" id="rationaleText"></p>
        </div>

        <!-- Confidence + scores meta -->
        <div class="meta-row">
          <div class="meta-item">
            <span class="meta-label">CONFIDENCE</span>
            <span class="meta-value" id="rConfidence"></span>
          </div>
          <div class="meta-item">
            <span class="meta-label">OPPORTUNITY SCORE</span>
            <span class="meta-value" id="rOpportunity"></span>
          </div>
          <div class="meta-item">
            <span class="meta-label">ADJUSTED SCORE</span>
            <span class="meta-value" id="rAdjusted"></span>
          </div>
        </div>

        <button class="btn-secondary" id="btnReset">START NEW DECISION</button>
      </div>
    </section>

  </div><!-- /app-shell -->

  <!-- Loading overlay -->
  <div class="loading-overlay hidden" id="loadingOverlay">
    <div class="loader-inner">
      <div class="loader-spinner"></div>
      <p class="loader-text" id="loaderText">PROCESSING...</p>
    </div>
  </div>

  <!-- Toast notification -->
  <div class="toast hidden" id="toast"></div>

  <script src="/js/app.js"></script>
</body>
</html>
/* ============================================================
   DECISION HELPER — app.css
   Aesthetic: Brutalist Precision — dark terminal meets
              high-end data dashboard. Think mission control.
   Fonts: Syne (display/UI) + JetBrains Mono (data/code)
   ============================================================ */

/* ── 1. DESIGN TOKENS ──────────────────────────────────────── */
:root {
  --bg:          #080b0f;
  --bg-2:        #0d1117;
  --bg-3:        #131922;
  --border:      #1e2a38;
  --border-hi:   #2a3f55;
  --accent:      #00e5ff;
  --accent-dim:  #00aacc;
  --accent-glow: rgba(0,229,255,0.12);
  --warn:        #ff9f43;
  --danger:      #ff4757;
  --success:     #2ed573;
  --text-hi:     #f0f4f8;
  --text-mid:    #8ea8c3;
  --text-lo:     #3d5468;
  --font-ui:     'Syne', sans-serif;
  --font-mono:   'JetBrains Mono', monospace;
  --ease-out:    cubic-bezier(0.16, 1, 0.3, 1);
  --radius:      4px;
}

/* ── 2. RESET ─────────────────────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html, body {
  height: 100%;
  background: var(--bg);
  color: var(--text-hi);
  font-family: var(--font-ui);
  overflow: hidden;
}

/* ── 3. GRID TEXTURE ──────────────────────────────────────── */
.grid-overlay {
  position: fixed;
  inset: 0;
  background-image:
    linear-gradient(rgba(0,229,255,0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0,229,255,0.03) 1px, transparent 1px);
  background-size: 40px 40px;
  pointer-events: none;
  z-index: 0;
}

/* ── 4. APP SHELL ─────────────────────────────────────────── */
.app-shell {
  position: relative;
  z-index: 1;
  height: 100vh;
  overflow: hidden;
}

/* ── 5. SCREENS ───────────────────────────────────────────── */
.screen {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  opacity: 0;
  pointer-events: none;
  transform: translateY(20px);
  transition:
    opacity 0.45s var(--ease-out),
    transform 0.45s var(--ease-out);
  overflow-y: auto;
}
.screen[data-active="true"] {
  opacity: 1;
  pointer-events: all;
  transform: translateY(0);
}

/* ── 6. SCREEN 1: INTRO ───────────────────────────────────── */
.intro-wrap {
  width: 100%;
  max-width: 640px;
  display: flex;
  flex-direction: column;
  gap: 3rem;
  animation: slide-in 0.7s var(--ease-out) both;
}

.logo-block {
  display: flex;
  align-items: flex-start;
  gap: 1.2rem;
}
.logo-mark {
  font-size: 3rem;
  color: var(--accent);
  line-height: 1;
  text-shadow: 0 0 30px var(--accent);
  flex-shrink: 0;
  animation: pulse-glow 3s ease-in-out infinite;
}
.logo-title {
  font-family: var(--font-ui);
  font-size: clamp(2.2rem, 5vw, 3.5rem);
  font-weight: 800;
  line-height: 0.95;
  letter-spacing: -0.02em;
  color: var(--text-hi);
}
.logo-sub {
  font-family: var(--font-mono);
  font-size: 0.68rem;
  color: var(--text-lo);
  letter-spacing: 0.15em;
  margin-top: 0.4rem;
}

.intro-body {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}
.intro-desc {
  font-size: 0.95rem;
  color: var(--text-mid);
  line-height: 1.7;
  border-left: 2px solid var(--border-hi);
  padding-left: 1rem;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.field-label {
  font-family: var(--font-mono);
  font-size: 0.65rem;
  letter-spacing: 0.18em;
  color: var(--accent);
}
.text-input {
  width: 100%;
  background: var(--bg-2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 0.85rem 1rem;
  color: var(--text-hi);
  font-family: var(--font-ui);
  font-size: 1rem;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s;
}
.text-input:focus {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px var(--accent-glow);
}
.text-input::placeholder { color: var(--text-lo); }
.char-count {
  font-family: var(--font-mono);
  font-size: 0.65rem;
  color: var(--text-lo);
  text-align: right;
}

.system-info {
  font-family: var(--font-mono);
  font-size: 0.62rem;
  letter-spacing: 0.15em;
  color: var(--text-lo);
  display: flex;
  gap: 0.6rem;
  flex-wrap: wrap;
}
.sep { color: var(--border-hi); }

/* ── 7. BUTTONS ───────────────────────────────────────────── */
.btn-primary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  background: var(--accent);
  color: var(--bg);
  font-family: var(--font-ui);
  font-size: 0.85rem;
  font-weight: 700;
  letter-spacing: 0.12em;
  padding: 0.9rem 1.4rem;
  border: none;
  border-radius: var(--radius);
  cursor: pointer;
  transition: background 0.2s, transform 0.2s var(--ease-out), box-shadow 0.2s;
}
.btn-primary:hover:not(:disabled) {
  background: #33eeff;
  box-shadow: 0 0 24px rgba(0,229,255,0.35);
  transform: translateY(-2px);
}
.btn-primary:active:not(:disabled) { transform: translateY(0); }
.btn-primary:disabled {
  background: var(--bg-3);
  color: var(--text-lo);
  cursor: not-allowed;
}
.btn-arrow { font-size: 1.1rem; }

.btn-secondary {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  background: transparent;
  color: var(--accent);
  font-family: var(--font-ui);
  font-size: 0.8rem;
  font-weight: 700;
  letter-spacing: 0.12em;
  padding: 0.75rem 1.2rem;
  border: 1px solid var(--accent);
  border-radius: var(--radius);
  cursor: pointer;
  transition: background 0.2s, transform 0.2s var(--ease-out);
}
.btn-secondary:hover {
  background: var(--accent-glow);
  transform: translateY(-2px);
}

/* ── 8. SCREEN 2: QUESTIONS ───────────────────────────────── */
.q-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 10;
  padding: 1rem 2rem;
  background: rgba(8,11,15,0.9);
  backdrop-filter: blur(16px);
  border-bottom: 1px solid var(--border);
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
}
.q-meta { display: flex; flex-direction: column; gap: 0.15rem; }
.session-label {
  font-family: var(--font-mono);
  font-size: 0.6rem;
  letter-spacing: 0.2em;
  color: var(--accent);
}
.decision-title-display {
  font-size: 0.85rem;
  font-weight: 700;
  color: var(--text-hi);
  max-width: 300px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.progress-wrap { display: flex; align-items: center; gap: 1rem; }
.progress-bar-track {
  width: 160px;
  height: 3px;
  background: var(--bg-3);
  border-radius: 999px;
  overflow: hidden;
}
.progress-bar-fill {
  height: 100%;
  background: var(--accent);
  border-radius: 999px;
  width: 0%;
  transition: width 0.6s var(--ease-out);
  box-shadow: 0 0 8px var(--accent);
}
.progress-text {
  font-family: var(--font-mono);
  font-size: 0.7rem;
  color: var(--text-mid);
  white-space: nowrap;
}

#screen-questions .screen { align-items: flex-start; }
.q-main {
  margin-top: 6rem;
  width: 100%;
  max-width: 680px;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  padding-bottom: 4rem;
  animation: slide-in 0.5s var(--ease-out) both;
}

.q-category-tag {
  font-family: var(--font-mono);
  font-size: 0.62rem;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: var(--accent);
  background: var(--accent-glow);
  border: 1px solid rgba(0,229,255,0.2);
  display: inline-block;
  padding: 0.25rem 0.7rem;
  border-radius: var(--radius);
}

.q-number {
  font-family: var(--font-mono);
  font-size: 0.68rem;
  color: var(--text-lo);
  letter-spacing: 0.1em;
}

.q-text {
  font-size: clamp(1.3rem, 3vw, 1.75rem);
  font-weight: 700;
  line-height: 1.35;
  color: var(--text-hi);
  letter-spacing: -0.01em;
}

.q-hint {
  font-family: var(--font-mono);
  font-size: 0.72rem;
  color: var(--text-mid);
  line-height: 1.6;
  border-left: 2px solid var(--border-hi);
  padding-left: 0.8rem;
}

/* Scale input (1-10) */
.scale-wrap {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.scale-labels {
  display: flex;
  justify-content: space-between;
  font-family: var(--font-mono);
  font-size: 0.62rem;
  color: var(--text-lo);
}
.scale-slider {
  width: 100%;
  -webkit-appearance: none;
  height: 4px;
  background: var(--bg-3);
  border-radius: 999px;
  outline: none;
  cursor: pointer;
}
.scale-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: var(--accent);
  box-shadow: 0 0 12px var(--accent);
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}
.scale-slider::-webkit-slider-thumb:hover {
  transform: scale(1.25);
  box-shadow: 0 0 20px var(--accent);
}
.scale-value-display {
  font-family: var(--font-mono);
  font-size: 2.5rem;
  font-weight: 500;
  color: var(--accent);
  text-align: center;
  text-shadow: 0 0 20px var(--accent);
  letter-spacing: -0.02em;
}

/* Choice buttons */
.choice-grid {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.choice-btn {
  width: 100%;
  text-align: left;
  padding: 0.9rem 1.1rem;
  background: var(--bg-2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: var(--text-mid);
  font-family: var(--font-ui);
  font-size: 0.88rem;
  cursor: pointer;
  transition: border-color 0.2s, color 0.2s, background 0.2s, transform 0.15s;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.choice-btn:hover {
  border-color: var(--border-hi);
  color: var(--text-hi);
  transform: translateX(4px);
}
.choice-btn.selected {
  border-color: var(--accent);
  background: var(--accent-glow);
  color: var(--text-hi);
}
.choice-btn.selected::after { content: '✓'; color: var(--accent); font-size: 0.9rem; }

/* Weight indicator */
.q-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 1rem;
  margin-top: 0.5rem;
}
.q-weight-indicator {
  font-family: var(--font-mono);
  font-size: 0.62rem;
  color: var(--text-lo);
  letter-spacing: 0.1em;
}

/* ── 9. SCREEN 3: RESULT ──────────────────────────────────── */
.result-wrap {
  width: 100%;
  max-width: 700px;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  padding: 7rem 0 4rem;
  animation: slide-in 0.6s var(--ease-out) both;
}

.result-header { display: flex; flex-direction: column; gap: 0.4rem; }
.result-label {
  font-family: var(--font-mono);
  font-size: 0.62rem;
  letter-spacing: 0.22em;
  color: var(--accent);
}
.result-decision-name {
  font-size: clamp(1.4rem, 3.5vw, 2rem);
  font-weight: 800;
  letter-spacing: -0.02em;
  color: var(--text-hi);
}

/* Verdict block */
.verdict-block {
  padding: 1.8rem;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  background: var(--bg-2);
  display: flex;
  flex-direction: column;
  gap: 0.7rem;
  border-left: 4px solid var(--accent);
}
.verdict-block.verdict-proceed      { border-left-color: var(--success); }
.verdict-block.verdict-caution      { border-left-color: var(--warn); }
.verdict-block.verdict-defer        { border-left-color: var(--warn); }
.verdict-block.verdict-no-proceed   { border-left-color: var(--danger); }

.verdict-label {
  font-family: var(--font-mono);
  font-size: 0.62rem;
  letter-spacing: 0.2em;
  color: var(--text-lo);
}
.verdict-text {
  font-size: clamp(1.4rem, 4vw, 2.2rem);
  font-weight: 800;
  letter-spacing: -0.01em;
  color: var(--text-hi);
}
.verdict-meta {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: var(--text-mid);
}

/* Score grid */
.score-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(130px, 1fr));
  gap: 0.75rem;
}
.score-card {
  background: var(--bg-2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}
.score-card-label {
  font-family: var(--font-mono);
  font-size: 0.6rem;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  color: var(--text-lo);
}
.score-card-bar-track {
  height: 3px;
  background: var(--bg-3);
  border-radius: 999px;
  overflow: hidden;
}
.score-card-bar-fill {
  height: 100%;
  border-radius: 999px;
  transition: width 1s 0.3s var(--ease-out);
}
.score-card-value {
  font-family: var(--font-mono);
  font-size: 1.5rem;
  font-weight: 500;
  color: var(--text-hi);
}

/* Risk panel */
.risk-panel {
  background: var(--bg-2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1.2rem;
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}
.risk-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.risk-title {
  font-family: var(--font-mono);
  font-size: 0.62rem;
  letter-spacing: 0.2em;
  color: var(--text-lo);
}
.risk-badge {
  font-family: var(--font-mono);
  font-size: 0.65rem;
  font-weight: 500;
  letter-spacing: 0.1em;
  padding: 0.2rem 0.6rem;
  border-radius: var(--radius);
}
.risk-badge.LOW      { background: rgba(46,213,115,0.12); color: var(--success); border: 1px solid rgba(46,213,115,0.3); }
.risk-badge.MEDIUM   { background: rgba(255,159,67,0.12);  color: var(--warn);    border: 1px solid rgba(255,159,67,0.3); }
.risk-badge.HIGH     { background: rgba(255,71,87,0.12);   color: var(--danger);  border: 1px solid rgba(255,71,87,0.3); }
.risk-badge.CRITICAL { background: rgba(255,71,87,0.2);    color: var(--danger);  border: 1px solid var(--danger); animation: pulse-danger 1s ease-in-out infinite; }

.risk-items { display: flex; flex-direction: column; gap: 0.5rem; }
.risk-item {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  color: var(--text-mid);
  padding: 0.5rem 0.7rem;
  border-radius: var(--radius);
  display: flex;
  align-items: flex-start;
  gap: 0.6rem;
}
.risk-item.flag    { background: rgba(255,71,87,0.07);   color: var(--danger);  }
.risk-item.warning { background: rgba(255,159,67,0.07);  color: var(--warn);    }
.risk-item.ok      { background: rgba(46,213,115,0.07);  color: var(--success); }
.risk-icon { flex-shrink: 0; }

/* Action + Rationale blocks */
.action-block, .rationale-block {
  background: var(--bg-2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1.2rem;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}
.action-label, .rationale-label {
  font-family: var(--font-mono);
  font-size: 0.6rem;
  letter-spacing: 0.2em;
  color: var(--text-lo);
}
.action-text {
  font-size: 0.95rem;
  font-weight: 700;
  color: var(--text-hi);
  line-height: 1.5;
}
.rationale-text {
  font-size: 0.88rem;
  color: var(--text-mid);
  line-height: 1.7;
}

/* Meta row */
.meta-row {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
}
.meta-item {
  flex: 1;
  min-width: 120px;
  background: var(--bg-2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 0.9rem;
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}
.meta-label {
  font-family: var(--font-mono);
  font-size: 0.58rem;
  letter-spacing: 0.15em;
  color: var(--text-lo);
}
.meta-value {
  font-family: var(--font-mono);
  font-size: 1.1rem;
  font-weight: 500;
  color: var(--accent);
}

/* ── 10. LOADING OVERLAY ──────────────────────────────────── */
.loading-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  background: rgba(8,11,15,0.85);
  backdrop-filter: blur(8px);
  display: flex;
  align-items: center;
  justify-content: center;
}
.loader-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}
.loader-spinner {
  width: 40px;
  height: 40px;
  border: 2px solid var(--bg-3);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
.loader-text {
  font-family: var(--font-mono);
  font-size: 0.7rem;
  letter-spacing: 0.2em;
  color: var(--accent);
}

/* ── 11. TOAST ────────────────────────────────────────────── */
.toast {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 200;
  background: var(--bg-3);
  border: 1px solid var(--border-hi);
  color: var(--text-hi);
  font-family: var(--font-mono);
  font-size: 0.75rem;
  padding: 0.7rem 1.2rem;
  border-radius: var(--radius);
  letter-spacing: 0.08em;
  animation: toast-in 0.3s var(--ease-out) both;
}
.toast.error { border-color: var(--danger); color: var(--danger); }

/* ── 12. ANIMATIONS ───────────────────────────────────────── */
@keyframes slide-in {
  from { opacity: 0; transform: translateY(16px); }
  to   { opacity: 1; transform: translateY(0); }
}
@keyframes spin {
  to { transform: rotate(360deg); }
}
@keyframes pulse-glow {
  0%,100% { text-shadow: 0 0 20px var(--accent); }
  50%      { text-shadow: 0 0 40px var(--accent), 0 0 80px rgba(0,229,255,0.3); }
}
@keyframes pulse-danger {
  0%,100% { box-shadow: none; }
  50%      { box-shadow: 0 0 12px rgba(255,71,87,0.4); }
}
@keyframes toast-in {
  from { opacity: 0; transform: translate(-50%, 10px); }
  to   { opacity: 1; transform: translate(-50%, 0); }
}
@keyframes q-transition {
  from { opacity: 0; transform: translateX(20px); }
  to   { opacity: 1; transform: translateX(0); }
}
.q-animate { animation: q-transition 0.4s var(--ease-out) both; }

/* ── 13. UTILITIES ────────────────────────────────────────── */
.hidden { display: none !important; }

/* ── 14. RESPONSIVE ───────────────────────────────────────── */
@media (max-width: 600px) {
  .screen { padding: 1rem; }
  .intro-wrap, .q-main, .result-wrap { max-width: 100%; }
  .q-header { padding: 0.8rem 1rem; }
  .progress-bar-track { width: 100px; }
  .decision-title-display { max-width: 150px; }
  .score-grid { grid-template-columns: 1fr 1fr; }
}
/**
 * ============================================================
 * DECISION HELPER — public/js/app.js
 *
 * Frontend controller. Communicates with the backend API,
 * manages screen transitions, and renders all dynamic UI.
 *
 * Architecture:
 *   State → API calls → DOM updates
 *   No framework. Pure vanilla JS with async/await.
 * ============================================================
 */

'use strict';

/* ── 1. STATE ─────────────────────────────────────────────── */
const state = {
  sessionId:      null,
  questions:      [],
  currentIndex:   0,
  totalQuestions: 0,
  selectedValue:  null,  // current answer before submitting
};

/* ── 2. DOM REFS ──────────────────────────────────────────── */
const $ = id => document.getElementById(id);

const screens = {
  intro:     $('screen-intro'),
  questions: $('screen-questions'),
  result:    $('screen-result'),
};

const ui = {
  decisionTitle:    $('decisionTitle'),
  charCount:        $('charCount'),
  btnStart:         $('btnStart'),
  qDecisionTitle:   $('qDecisionTitle'),
  progressFill:     $('progressFill'),
  progressText:     $('progressText'),
  qCategory:        $('qCategory'),
  qNumber:          $('qNumber'),
  qText:            $('qText'),
  qHint:            $('qHint'),
  answerZone:       $('answerZone'),
  qWeightIndicator: $('qWeightIndicator'),
  btnNext:          $('btnNext'),
  loadingOverlay:   $('loadingOverlay'),
  loaderText:       $('loaderText'),
  toast:            $('toast'),
  // Result refs
  rDecisionTitle:   $('rDecisionTitle'),
  verdictBlock:     $('verdictBlock'),
  verdictText:      $('verdictText'),
  verdictMeta:      $('verdictMeta'),
  scoreGrid:        $('scoreGrid'),
  riskPanel:        $('riskPanel'),
  riskBadge:        $('riskBadge'),
  riskItems:        $('riskItems'),
  actionText:       $('actionText'),
  rationaleText:    $('rationaleText'),
  rConfidence:      $('rConfidence'),
  rOpportunity:     $('rOpportunity'),
  rAdjusted:        $('rAdjusted'),
  btnReset:         $('btnReset'),
};

/* ── 3. SCREEN MANAGER ────────────────────────────────────── */
function showScreen(name) {
  Object.entries(screens).forEach(([key, el]) => {
    el.setAttribute('data-active', key === name ? 'true' : 'false');
  });
  // Scroll question/result screen to top
  if (name === 'questions' || name === 'result') {
    screens[name].scrollTop = 0;
  }
}

/* ── 4. API LAYER ─────────────────────────────────────────── */
const API_BASE = '/api/decision';

async function apiCall(endpoint, body) {
  const res = await fetch(`${API_BASE}${endpoint}`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(body),
  });
  const data = await res.json();
  if (!res.ok || !data.ok) {
    throw new Error(data.error || `Request failed (${res.status})`);
  }
  return data;
}

/* ── 5. LOADING HELPERS ───────────────────────────────────── */
function showLoading(text = 'PROCESSING...') {
  ui.loaderText.textContent = text;
  ui.loadingOverlay.classList.remove('hidden');
}
function hideLoading() {
  ui.loadingOverlay.classList.add('hidden');
}

/* ── 6. TOAST ─────────────────────────────────────────────── */
let toastTimer;
function showToast(msg, type = '') {
  clearTimeout(toastTimer);
  ui.toast.textContent = msg;
  ui.toast.className   = `toast${type ? ' ' + type : ''}`;
  ui.toast.classList.remove('hidden');
  toastTimer = setTimeout(() => ui.toast.classList.add('hidden'), 3500);
}

/* ── 7. INTRO SCREEN ──────────────────────────────────────── */
ui.decisionTitle.addEventListener('input', () => {
  const len = ui.decisionTitle.value.length;
  ui.charCount.textContent = `${len} / 120`;
  ui.btnStart.disabled     = len < 2;
});

ui.decisionTitle.addEventListener('keydown', e => {
  if (e.key === 'Enter' && !ui.btnStart.disabled) startDecision();
});

ui.btnStart.addEventListener('click', startDecision);

async function startDecision() {
  const title = ui.decisionTitle.value.trim();
  if (title.length < 2) return;

  try {
    showLoading('INITIALIZING SESSION...');
    const data = await apiCall('/start', { title });

    // Store state
    state.sessionId      = data.sessionId;
    state.questions      = data.questions;
    state.totalQuestions = data.totalQuestions;
    state.currentIndex   = 0;

    // Update header
    ui.qDecisionTitle.textContent = title;

    hideLoading();
    showScreen('questions');
    renderQuestion(0);

  } catch (err) {
    hideLoading();
    showToast(`⚠ ${err.message}`, 'error');
  }
}

/* ── 8. QUESTION RENDERING ────────────────────────────────── */
function renderQuestion(index) {
  const q = state.questions[index];
  if (!q) return;

  state.selectedValue = null;
  ui.btnNext.disabled = true;

  // Animate transition
  ui.qCategory.classList.remove('q-animate');
  void ui.qCategory.offsetWidth; // reflow
  ui.qCategory.classList.add('q-animate');

  // Category tag
  ui.qCategory.textContent = q.category.toUpperCase();

  // Question number
  ui.qNumber.textContent = `QUESTION ${index + 1} OF ${state.totalQuestions}`;

  // Question text
  ui.qText.textContent = q.text;
  ui.qText.classList.remove('q-animate');
  void ui.qText.offsetWidth;
  ui.qText.classList.add('q-animate');

  // Hint
  ui.qHint.textContent = q.hint || '';

  // Weight indicator
  const weightLabel = q.weight >= 1.8 ? '⬆ HIGH IMPACT'
                    : q.weight >= 1.3 ? '→ MEDIUM IMPACT'
                    : '↓ STANDARD WEIGHT';
  ui.qWeightIndicator.textContent = `WEIGHT ${q.weight}× — ${weightLabel}`;

  // Progress
  updateProgress(index, state.totalQuestions);

  // Render answer input
  renderAnswerZone(q);
}

function renderAnswerZone(q) {
  ui.answerZone.innerHTML = '';
  ui.answerZone.classList.remove('q-animate');
  void ui.answerZone.offsetWidth;
  ui.answerZone.classList.add('q-animate');

  if (q.type === 'scale') {
    renderScale(q);
  } else if (q.type === 'choice') {
    renderChoices(q);
  }
}

function renderScale(q) {
  const wrap = document.createElement('div');
  wrap.className = 'scale-wrap';

  const display = document.createElement('div');
  display.className = 'scale-value-display';
  display.textContent = '5';

  const labelsEl = document.createElement('div');
  labelsEl.className = 'scale-labels';
  labelsEl.innerHTML = `<span>${q.min}</span><span>${q.max}</span>`;

  const slider = document.createElement('input');
  slider.type      = 'range';
  slider.min       = q.min;
  slider.max       = q.max;
  slider.value     = 5;
  slider.className = 'scale-slider';

  slider.addEventListener('input', () => {
    display.textContent  = slider.value;
    state.selectedValue  = Number(slider.value);
    ui.btnNext.disabled  = false;
  });

  // Default selection at 5
  state.selectedValue = 5;
  ui.btnNext.disabled = false;

  wrap.appendChild(display);
  wrap.appendChild(labelsEl);
  wrap.appendChild(slider);
  ui.answerZone.appendChild(wrap);
}

function renderChoices(q) {
  const grid = document.createElement('div');
  grid.className = 'choice-grid';

  q.options.forEach(opt => {
    const btn = document.createElement('button');
    btn.className    = 'choice-btn';
    btn.textContent  = opt.label;
    btn.dataset.value = opt.value;

    btn.addEventListener('click', () => {
      grid.querySelectorAll('.choice-btn').forEach(b => b.classList.remove('selected'));
      btn.classList.add('selected');
      state.selectedValue = opt.value;
      ui.btnNext.disabled = false;
    });

    grid.appendChild(btn);
  });

  ui.answerZone.appendChild(grid);
}

function updateProgress(index, total) {
  const answered = index; // current question not yet answered
  const pct      = Math.round((answered / total) * 100);
  ui.progressFill.style.width = `${pct}%`;
  ui.progressText.textContent = `${answered} / ${total}`;
}

/* ── 9. NEXT BUTTON ───────────────────────────────────────── */
ui.btnNext.addEventListener('click', submitAnswer);

async function submitAnswer() {
  const q = state.questions[state.currentIndex];
  if (state.selectedValue === null) return;

  try {
    showLoading('RECORDING ANSWER...');

    const data = await apiCall('/answer', {
      sessionId:  state.sessionId,
      questionId: q.id,
      value:      state.selectedValue,
    });

    hideLoading();

    if (data.isComplete) {
      // All answered — fetch result
      await fetchResult();
    } else {
      state.currentIndex++;
      renderQuestion(state.currentIndex);
    }

  } catch (err) {
    hideLoading();
    showToast(`⚠ ${err.message}`, 'error');
  }
}

/* ── 10. RESULT FETCHING & RENDERING ─────────────────────── */
async function fetchResult() {
  try {
    showLoading('RUNNING ANALYSIS ENGINE...');
    const data = await apiCall('/result', { sessionId: state.sessionId });
    hideLoading();
    renderResult(data.result);
    showScreen('result');
  } catch (err) {
    hideLoading();
    showToast(`⚠ ${err.message}`, 'error');
  }
}

function renderResult(result) {
  // Title
  ui.rDecisionTitle.textContent = result.decisionTitle;

  // Verdict
  const rec = result.recommendation;
  ui.verdictText.textContent = rec.decision;
  ui.verdictMeta.textContent =
    `Opportunity: ${rec.opportunityScore}/100  ·  Adjusted: ${rec.adjustedScore}/100`;

  // Verdict block color class
  const verdictClass = rec.decision.includes('PROCEED') && !rec.decision.includes('CAUTION') ? 'verdict-proceed'
                     : rec.decision.includes('CAUTION') ? 'verdict-caution'
                     : rec.decision.includes('DEFER')   ? 'verdict-defer'
                     : 'verdict-no-proceed';
  ui.verdictBlock.className = `verdict-block ${verdictClass}`;

  // Score grid
  ui.scoreGrid.innerHTML = '';
  const CAT_COLORS = {
    feasibility: '#00e5ff',
    risk:        '#ff4757',
    impact:      '#2ed573',
    resources:   '#ff9f43',
    urgency:     '#a29bfe',
  };

  Object.entries(result.scores).forEach(([cat, score]) => {
    // For risk, display effective risk (inverted)
    const displayScore = cat === 'risk'
      ? result.riskReport.effectiveRiskScore
      : score;
    const displayLabel = cat === 'risk' ? 'risk exposure' : cat;
    const color = CAT_COLORS[cat] || '#00e5ff';

    const card = document.createElement('div');
    card.className = 'score-card';
    card.innerHTML = `
      <div class="score-card-label">${displayLabel.toUpperCase()}</div>
      <div class="score-card-bar-track">
        <div class="score-card-bar-fill"
          style="width:0%;background:${color};"
          data-target="${displayScore}"></div>
      </div>
      <div class="score-card-value">${displayScore}</div>
    `;
    ui.scoreGrid.appendChild(card);
  });

  // Animate bars after paint
  requestAnimationFrame(() => {
    document.querySelectorAll('.score-card-bar-fill').forEach(bar => {
      bar.style.width = `${bar.dataset.target}%`;
    });
  });

  // Risk panel
  const risk = result.riskReport;
  ui.riskBadge.textContent = risk.severity;
  ui.riskBadge.className   = `risk-badge ${risk.severity}`;

  ui.riskItems.innerHTML = '';
  if (risk.flags.length === 0 && risk.warnings.length === 0) {
    ui.riskItems.innerHTML = `<div class="risk-item ok"><span class="risk-icon">✓</span>No critical flags detected. Risk profile is within acceptable parameters.</div>`;
  }
  risk.flags.forEach(f => {
    const el = document.createElement('div');
    el.className = 'risk-item flag';
    el.innerHTML = `<span class="risk-icon">⚑</span>${f.message}`;
    ui.riskItems.appendChild(el);
  });
  risk.warnings.forEach(w => {
    const el = document.createElement('div');
    el.className = 'risk-item warning';
    el.innerHTML = `<span class="risk-icon">⚠</span>${w.message}`;
    ui.riskItems.appendChild(el);
  });

  // Action + Rationale
  ui.actionText.textContent    = rec.action;
  ui.rationaleText.textContent = rec.rationale;

  // Meta row
  ui.rConfidence.textContent  = `${result.confidence.score} — ${result.confidence.label}`;
  ui.rOpportunity.textContent = `${rec.opportunityScore}/100`;
  ui.rAdjusted.textContent    = `${rec.adjustedScore}/100`;
}

/* ── 11. RESET ────────────────────────────────────────────── */
ui.btnReset.addEventListener('click', () => {
  // Clear state
  state.sessionId      = null;
  state.questions      = [];
  state.currentIndex   = 0;
  state.totalQuestions = 0;
  state.selectedValue  = null;

  // Clear inputs
  ui.decisionTitle.value = '';
  ui.charCount.textContent = '0 / 120';
  ui.btnStart.disabled     = true;

  // Reset progress bar
  ui.progressFill.style.width = '0%';

  showScreen('intro');
});

/* ── 12. KEYBOARD NAV ─────────────────────────────────────── */
document.addEventListener('keydown', e => {
  // Enter on question screen advances if answer selected
  if (e.key === 'Enter' &&
      screens.questions.getAttribute('data-active') === 'true' &&
      !ui.btnNext.disabled) {
    submitAnswer();
  }
});
{
  "name": "decision-helper",
  "version": "1.0.0",
  "description": "Advanced Decision Logic Engine",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "uuid": "^9.0.0"
  }
}
