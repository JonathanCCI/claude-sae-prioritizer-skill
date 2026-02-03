# React Artifact Template

Production-ready React component with **weekly account rotation** using persistent storage. Chrome 143+ optimized.

## Complete Component

```jsx
import React, { useState, useCallback, useMemo, useRef, useEffect } from 'react';

// ============================================================================
// BROWSER COMPATIBILITY (Chrome 143+)
// ============================================================================

const MINIMUM_CHROME_VERSION = 143;
const chromeMatch = navigator.userAgent.match(/Chrome\/(\d+)/);
const chromeVersion = chromeMatch ? parseInt(chromeMatch[1], 10) : 0;
const isCompatibleBrowser = chromeVersion >= MINIMUM_CHROME_VERSION;

const ALLOWED_API_ENDPOINTS = Object.freeze(['https://api.anthropic.com/v1/messages']);

// ============================================================================
// CIRCLECI THEME
// ============================================================================

const THEME = Object.freeze({
  primary: '#00DB74',
  primaryDark: '#2E3C52',
  bgLight: '#EDEDED',
  bgDark: '#1C273A',
  bgCard: '#1C273A',
  secondary: '#D4D4D4',
  blue: '#0057FF',
  red: '#FC5656',
  amber: '#F59E0B',
  amberMuted: 'rgba(245, 158, 11, 0.15)',
  lavender: '#ECDBFF',
  lightGreen: '#6BDE91',
  yellow: '#FFF500',
  titleOnLight: '#1C273A',
  headlineOnLight: '#161616',
  bodyOnLight: '#343434',
  titleOnDark: '#00DB74',
  titleAltOnDark: '#EDEDED',
  headlineOnDark: '#FFFFFF',
  bodyOnDark: '#FAFAFA',
  focus: '#FC5656',
  focusMuted: 'rgba(252, 86, 86, 0.15)',
  target: '#0057FF',
  targetMuted: 'rgba(0, 87, 255, 0.15)',
  primaryMuted: 'rgba(0, 219, 116, 0.15)',
  primaryBorder: 'rgba(0, 219, 116, 0.3)',
  cardBorder: 'rgba(237, 237, 237, 0.12)',
});

const CRITERIA = Object.freeze([
  { id: 'need', label: 'CI/CD Need', weight: 0.30, icon: '⚙️' },
  { id: 'current', label: 'Current Solution', weight: 0.25, icon: '🔄' },
  { id: 'market', label: 'Market Timing', weight: 0.20, icon: '📈' },
  { id: 'news', label: 'Company News', weight: 0.15, icon: '📰' },
  { id: 'contacts', label: 'Reachable Contacts', weight: 0.10, icon: '👥' }
]);

const PERSONAS = Object.freeze([
  "Developer", "Developer Experience", "Platform Team", "Ops/Infra Engineer",
  "Engineering Manager", "Security Engineer", "Engineering Leader"
]);

// ============================================================================
// SECURITY UTILITIES
// ============================================================================

const cleanText = (text) => {
  if (!text || typeof text !== 'string') return text;
  return text.replace(/<\/?cite[^>]*>/gi, '').replace(/<\/?antml:cite[^>]*>/gi, '')
    .replace(/\[?\d+-\d+(?:,\d+-\d+)*\]?/g, '').replace(/<[^>]+>/g, '').trim();
};

const cleanObject = (obj, depth = 0) => {
  if (depth > 10) return obj;
  if (typeof obj === 'string') return cleanText(obj);
  if (Array.isArray(obj)) return obj.map(item => cleanObject(item, depth + 1));
  if (obj && typeof obj === 'object') {
    return Object.fromEntries(Object.entries(obj).map(([k, v]) => [k, cleanObject(v, depth + 1)]));
  }
  return obj;
};

const validateScore = (score) => {
  const num = Number(score);
  return !isNaN(num) ? Math.max(0, Math.min(10, num)) : 0;
};

const validateAnalysisResponse = (data) => {
  if (!data || typeof data !== 'object') return false;
  if (!Array.isArray(data.topAccounts)) return false;
  return data.topAccounts.every(a => a && typeof a.company === 'string' && typeof a.rank === 'number');
};

const safeFetch = async (url, options, timeoutMs = 120000) => {
  if (!ALLOWED_API_ENDPOINTS.some(endpoint => url.startsWith(endpoint))) {
    throw new Error('Invalid API endpoint');
  }
  const response = await fetch(url, { ...options, signal: AbortSignal.timeout(timeoutMs) });
  if (!response.ok) throw new Error(`API error: ${response.status}`);
  return response;
};

// ============================================================================
// PERSISTENT STORAGE HELPERS (Weekly Rotation)
// ============================================================================

const getStorageKey = (ownerId) => `sae-priorities:${ownerId}`;

const getWeekStart = (date = new Date()) => {
  const d = new Date(date);
  const day = d.getDay();
  const diff = d.getDate() - day + (day === 0 ? -6 : 1); // Monday
  d.setDate(diff);
  d.setHours(0, 0, 0, 0);
  return d.toISOString().split('T')[0];
};

const loadPreviousWeek = async (ownerId) => {
  try {
    const result = await window.storage.get(getStorageKey(ownerId));
    if (result?.value) {
      const data = JSON.parse(result.value);
      const currentWeek = getWeekStart();
      // Only use if from a different week
      if (data.weekOf && data.weekOf !== currentWeek) {
        return data;
      }
    }
  } catch (e) {
    console.log('No previous week data or storage unavailable');
  }
  return null;
};

const saveCurrentWeek = async (ownerId, accountIds) => {
  try {
    const data = {
      weekOf: getWeekStart(),
      accountIds,
      analysisDate: new Date().toISOString().split('T')[0]
    };
    await window.storage.set(getStorageKey(ownerId), JSON.stringify(data));
    return true;
  } catch (e) {
    console.error('Failed to save week data:', e);
    return false;
  }
};

// ============================================================================
// DATA INJECTION POINTS
// ============================================================================

// CLAUDE: Replace with live Slack data
const VERIFIED_SAES = Object.freeze([
  /*__SAE_DATA_INJECTION_POINT__*/
]);

// CLAUDE: Replace with live HubSpot data
const PREFETCHED_ACCOUNTS = {
  /*__ACCOUNTS_DATA_INJECTION_POINT__*/
};

// ============================================================================
// MAIN COMPONENT
// ============================================================================

export default function SAEAccountPrioritizer() {
  if (!isCompatibleBrowser) {
    return (
      <div style={{ padding: '40px', textAlign: 'center', fontFamily: 'system-ui', background: THEME.bgLight, minHeight: '100vh' }}>
        <h2 style={{ color: THEME.red }}>Chrome 143+ Required</h2>
        <p>Current version: {chromeVersion || 'Unknown'}</p>
      </div>
    );
  }

  const [step, setStep] = useState('select-user');
  const [selectedSAE, setSelectedSAE] = useState(null);
  const [accounts, setAccounts] = useState([]);
  const [progress, setProgress] = useState({ current: 0, total: 0, message: '' });
  const [results, setResults] = useState(null);
  const [previousWeekData, setPreviousWeekData] = useState(null);
  const [error, setError] = useState(null);
  const [expandedCard, setExpandedCard] = useState(null);
  const [storageEnabled, setStorageEnabled] = useState(true);
  const abortControllerRef = useRef(null);

  useEffect(() => () => abortControllerRef.current?.abort(), []);

  const matchedSAEs = useMemo(() => VERIFIED_SAES.map(sae => ({ ...sae, hubspotMatch: true })), []);

  const fetchAccountsForSAE = useCallback(async (sae) => {
    if (!sae?.hubspotOwnerId) {
      setError(`${sae?.name || 'User'} has no HubSpot owner ID.`);
      return;
    }
    const saeAccounts = PREFETCHED_ACCOUNTS[sae.hubspotOwnerId];
    if (Array.isArray(saeAccounts) && saeAccounts.length > 0) {
      setAccounts(saeAccounts);
      // Load previous week data for rotation
      const prevData = await loadPreviousWeek(sae.hubspotOwnerId);
      setPreviousWeekData(prevData);
      setStep('ready');
    } else {
      setError(`No accounts for ${sae.name}.`);
    }
  }, []);

  const handleSAESelect = useCallback((sae) => {
    setSelectedSAE(sae);
    setAccounts([]);
    setResults(null);
    setPreviousWeekData(null);
    setExpandedCard(null);
    setError(null);
    fetchAccountsForSAE(sae);
  }, [fetchAccountsForSAE]);

  const analyzeAccounts = useCallback(async () => {
    if (accounts.length === 0) return;
    setStep('analyzing');
    setError(null);
    setProgress({ current: 0, total: 4, message: '' });

    try {
      abortControllerRef.current?.abort();
      abortControllerRef.current = new AbortController();

      const accountNames = accounts.map(a => a.name);
      const focusAccounts = accounts.filter(a => a.isFocusAccount).map(a => a.name);
      const targetOnlyAccounts = accounts.filter(a => !a.isFocusAccount && a.isTargetAccount).map(a => a.name);

      // Note previous week accounts in prompt for context
      const previousAccountNames = previousWeekData?.accountIds 
        ? accounts.filter(a => previousWeekData.accountIds.includes(a.id)).map(a => a.name)
        : [];

      const analysisPrompt = `You are a sales intelligence analyst for CircleCI. Analyze these ${accounts.length} companies and score ALL of them for outreach priority.

COMPANIES (owned by ${selectedSAE.name}):
${accountNames.map((c, i) => `${i + 1}. ${c}`).join('\n')}

FOCUS ACCOUNTS (Big Bet): ${focusAccounts.length > 0 ? focusAccounts.join(', ') : 'None'}
TARGET ACCOUNTS: ${targetOnlyAccounts.length > 0 ? targetOnlyAccounts.join(', ') : 'None'}
${previousAccountNames.length > 0 ? `\nLAST WEEK'S TOP 10 (for context): ${previousAccountNames.join(', ')}` : ''}

TARGET PERSONAS: ${PERSONAS.join(', ')}

SCORING (1-10 each):
1. CI/CD NEED (30%): Engineering complexity, deployment needs
2. CURRENT SOLUTION (25%): Competitor tools, migration potential  
3. MARKET TIMING (20%): GitHub Actions pricing, Jenkins migration trends
4. COMPANY NEWS (15%): Funding, growth, tech leadership changes
5. REACHABLE CONTACTS (10%): DevOps/Platform personas identifiable

Return JSON with TOP 15 accounts (we need extras for rotation logic):
{"marketContext":"2-3 sentences","analysisDate":"YYYY-MM-DD","ownerName":"${selectedSAE.name}","totalAccountsAnalyzed":${accounts.length},"topAccounts":[{"rank":1,"company":"Name","accountId":"HubSpot ID if known","isFocusAccount":true,"scores":{"need":8,"current":7,"market":9,"news":6,"contacts":8},"totalScore":7.8,"summary":"Rationale","currentCICD":"Tools","keyTriggers":["Trigger"],"targetPersonas":["VP Engineering"],"outreachAngle":"Opener","sources":[{"title":"Title","url":"https://..."}]}]}

Return raw JSON only, no markdown.`;

      setProgress({ current: 1, total: 4, message: 'Searching market intelligence...' });

      const response = await safeFetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 10000,
          tools: [{ type: "web_search_20250305", name: "web_search" }],
          messages: [{ role: "user", content: analysisPrompt }]
        })
      }, 150000);

      setProgress({ current: 2, total: 4, message: 'Processing recommendations...' });
      const data = await response.json();

      let textContent = '';
      for (const block of data.content || []) {
        if (block.type === 'text') textContent += block.text;
      }

      const jsonMatch = textContent.match(/\{[\s\S]*\}/);
      if (!jsonMatch) throw new Error('No valid JSON in response');

      const parsed = JSON.parse(jsonMatch[0].replace(/```json\s*/gi, '').replace(/```\s*/gi, ''));
      if (!validateAnalysisResponse(parsed)) throw new Error('Invalid response structure');

      const cleanedResults = cleanObject(parsed);
      
      // Validate scores
      if (cleanedResults.topAccounts) {
        cleanedResults.topAccounts = cleanedResults.topAccounts.map(account => ({
          ...account,
          scores: account.scores ? {
            need: validateScore(account.scores.need),
            current: validateScore(account.scores.current),
            market: validateScore(account.scores.market),
            news: validateScore(account.scores.news),
            contacts: validateScore(account.scores.contacts)
          } : account.scores
        }));
      }

      setProgress({ current: 3, total: 4, message: 'Applying weekly rotation...' });

      // === WEEKLY ROTATION LOGIC ===
      const allRanked = cleanedResults.topAccounts || [];
      const previousIds = new Set(previousWeekData?.accountIds || []);
      
      // Match account IDs from our data
      const enrichedAccounts = allRanked.map(account => {
        const matchedAccount = accounts.find(a => 
          a.name.toLowerCase() === account.company.toLowerCase() ||
          a.name.toLowerCase().includes(account.company.toLowerCase()) ||
          account.company.toLowerCase().includes(a.name.toLowerCase())
        );
        return {
          ...account,
          accountId: matchedAccount?.id || account.accountId || account.company,
          isFocusAccount: matchedAccount?.isFocusAccount ?? account.isFocusAccount,
          isTargetAccount: matchedAccount?.isTargetAccount ?? true
        };
      });

      // Split: NEW (not in last week) vs CONTINUED (was in last week)
      const newAccounts = [];
      const continuedAccounts = [];

      for (const account of enrichedAccounts) {
        if (previousIds.has(account.accountId)) {
          continuedAccounts.push({ ...account, isContinued: true });
        } else {
          newAccounts.push({ ...account, isContinued: false });
        }
      }

      // Take top 10 NEW accounts
      const top10New = newAccounts.slice(0, 10).map((a, i) => ({ ...a, rank: i + 1 }));
      
      // Continued accounts that would have been in top 10 → show as 11, 12, etc.
      const continuedHighPriority = continuedAccounts
        .filter(a => (a.totalScore || 0) >= (top10New[9]?.totalScore || 0))
        .map((a, i) => ({ 
          ...a, 
          rank: 11 + i,
          continuedReason: a.keyTriggers?.[0] || a.summary || 'Still high priority based on scoring'
        }));

      cleanedResults.topAccounts = [...top10New, ...continuedHighPriority];
      cleanedResults.newAccountCount = top10New.length;
      cleanedResults.continuedAccountCount = continuedHighPriority.length;
      cleanedResults.previousWeekOf = previousWeekData?.weekOf || null;

      // Save this week's top 10 NEW account IDs
      setProgress({ current: 4, total: 4, message: 'Saving for next week...' });
      const newIds = top10New.map(a => a.accountId);
      const saved = await saveCurrentWeek(selectedSAE.hubspotOwnerId, newIds);
      setStorageEnabled(saved);

      setResults(cleanedResults);
      setStep('results');
    } catch (err) {
      console.error('Analysis error:', err);
      setError(err.name === 'AbortError' ? 'Request cancelled' : `Analysis failed: ${err.message}`);
      setStep('ready');
    }
  }, [accounts, selectedSAE, previousWeekData]);

  const calculateWeightedScore = useCallback((scores) => {
    if (!scores || typeof scores !== 'object') return 0;
    return CRITERIA.reduce((sum, c) => sum + validateScore(scores[c.id]) * c.weight, 0);
  }, []);

  const getScoreColor = useCallback((rank, total = 10) => {
    const t = Math.min((rank - 1) / Math.max(total - 1, 1), 1);
    return `rgb(${Math.round(229 - t * 196)}, ${Math.round(57 + t * 93)}, ${Math.round(53 + t * 190)})`;
  }, []);

  const sortedResults = useMemo(() => {
    if (!results?.topAccounts) return [];
    return structuredClone(results.topAccounts).sort((a, b) => a.rank - b.rank);
  }, [results]);

  // ============================================================================
  // RENDER
  // ============================================================================

  return (
    <div style={{ minHeight: '100vh', background: THEME.bgLight, padding: '24px' }}>
      <div style={{ maxWidth: '1200px', margin: '0 auto' }}>
        <h1 style={{ fontFamily: "'Space Grotesk', sans-serif", color: THEME.titleOnLight, fontSize: '28px', marginBottom: '24px' }}>
          <span style={{ color: THEME.primary }}>●</span> SAE Account Prioritizer
        </h1>

        {error && (
          <div style={{ background: THEME.focusMuted, border: `1px solid ${THEME.focus}`, padding: '16px', borderRadius: '12px', marginBottom: '20px' }}>
            <span style={{ color: THEME.focus, fontWeight: 600 }}>Error:</span>
            <span style={{ color: THEME.bodyOnLight, marginLeft: '8px' }}>{error}</span>
          </div>
        )}

        {/* SELECT USER */}
        {step === 'select-user' && (
          <div>
            <h2 style={{ fontFamily: "'Space Grotesk', sans-serif", color: THEME.headlineOnLight, marginBottom: '16px' }}>Select SAE</h2>
            <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(280px, 1fr))', gap: '16px' }}>
              {matchedSAEs.map(sae => (
                <button key={sae.slackId} onClick={() => handleSAESelect(sae)} style={{
                  background: THEME.bgCard, border: `1px solid ${THEME.cardBorder}`, borderRadius: '12px',
                  padding: '16px', cursor: 'pointer', textAlign: 'left'
                }}>
                  <div style={{ color: THEME.headlineOnDark, fontWeight: 600, fontSize: '16px' }}>{sae.name}</div>
                  <div style={{ color: THEME.bodyOnDark, fontSize: '13px', marginTop: '4px' }}>{sae.title}</div>
                  <div style={{ color: THEME.primary, fontSize: '12px', marginTop: '8px' }}>{sae.email}</div>
                </button>
              ))}
            </div>
          </div>
        )}

        {/* READY */}
        {step === 'ready' && selectedSAE && (
          <div style={{ maxWidth: '800px', margin: '0 auto' }}>
            <div style={{ background: THEME.bgCard, borderRadius: '16px', padding: '24px' }}>
              <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'start', marginBottom: '20px' }}>
                <div>
                  <h2 style={{ fontFamily: "'Space Grotesk', sans-serif", color: THEME.titleOnDark, margin: '0 0 8px' }}>{selectedSAE.name}</h2>
                  <p style={{ color: THEME.bodyOnDark, margin: 0 }}>{selectedSAE.title}</p>
                </div>
                <button onClick={() => { setStep('select-user'); setSelectedSAE(null); setAccounts([]); }} style={{
                  background: 'transparent', border: `1px solid ${THEME.primaryBorder}`, color: THEME.bodyOnDark,
                  padding: '8px 16px', borderRadius: '8px', cursor: 'pointer'
                }}>← Back</button>
              </div>
              <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: '16px', marginBottom: '24px' }}>
                <div style={{ background: THEME.bgDark, padding: '16px', borderRadius: '12px', textAlign: 'center' }}>
                  <div style={{ color: THEME.primary, fontSize: '32px', fontWeight: 700 }}>{accounts.length}</div>
                  <div style={{ color: THEME.bodyOnDark, fontSize: '13px' }}>Total</div>
                </div>
                <div style={{ background: THEME.bgDark, padding: '16px', borderRadius: '12px', textAlign: 'center' }}>
                  <div style={{ color: THEME.focus, fontSize: '32px', fontWeight: 700 }}>{accounts.filter(a => a.isFocusAccount).length}</div>
                  <div style={{ color: THEME.bodyOnDark, fontSize: '13px' }}>Focus</div>
                </div>
                <div style={{ background: THEME.bgDark, padding: '16px', borderRadius: '12px', textAlign: 'center' }}>
                  <div style={{ color: THEME.target, fontSize: '32px', fontWeight: 700 }}>{accounts.filter(a => !a.isFocusAccount).length}</div>
                  <div style={{ color: THEME.bodyOnDark, fontSize: '13px' }}>Target</div>
                </div>
              </div>
              {previousWeekData && (
                <div style={{ background: THEME.amberMuted, border: `1px solid ${THEME.amber}`, padding: '12px 16px', borderRadius: '8px', marginBottom: '16px' }}>
                  <span style={{ color: THEME.amber }}>📅</span>
                  <span style={{ color: THEME.bodyOnLight, marginLeft: '8px' }}>
                    Last prioritized: {previousWeekData.weekOf} — Will rotate to show different accounts
                  </span>
                </div>
              )}
              <button onClick={analyzeAccounts} style={{
                width: '100%', padding: '16px', background: THEME.primary, color: THEME.bgDark, border: 'none',
                borderRadius: '12px', fontSize: '16px', fontWeight: 600, cursor: 'pointer'
              }}>🔍 Prioritize This Week</button>
            </div>
          </div>
        )}

        {/* ANALYZING */}
        {step === 'analyzing' && (
          <div style={{ maxWidth: '600px', margin: '0 auto' }}>
            <div style={{ background: THEME.bgCard, borderRadius: '16px', padding: '40px', textAlign: 'center' }}>
              <div style={{ fontSize: '48px', marginBottom: '20px', animation: 'pulse 2s infinite' }}>🔍</div>
              <h2 style={{ fontFamily: "'Space Grotesk', sans-serif", color: THEME.titleOnDark, margin: '0 0 12px' }}>Analyzing</h2>
              <p style={{ color: THEME.bodyOnDark, margin: '0 0 24px' }}>{progress.message}</p>
              <div style={{ background: THEME.bgDark, borderRadius: '8px', height: '8px', overflow: 'hidden' }}>
                <div style={{ background: THEME.primary, height: '100%', width: `${(progress.current / progress.total) * 100}%`, transition: 'width 0.5s' }} />
              </div>
              <p style={{ color: THEME.bodyOnDark, fontSize: '13px', marginTop: '12px' }}>Step {progress.current}/{progress.total}</p>
            </div>
          </div>
        )}

        {/* RESULTS */}
        {step === 'results' && results && (
          <div>
            <div style={{ background: THEME.bgCard, borderRadius: '16px', padding: '24px', marginBottom: '20px' }}>
              <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'start', flexWrap: 'wrap', gap: '16px' }}>
                <div>
                  <h2 style={{ fontFamily: "'Space Grotesk', sans-serif", color: THEME.titleOnDark, margin: '0 0 8px' }}>
                    Weekly priorities for {results.ownerName}
                  </h2>
                  <p style={{ color: THEME.bodyOnDark, margin: 0 }}>
                    {results.analysisDate} • {results.totalAccountsAnalyzed} analyzed • 
                    <span style={{ color: THEME.primary }}> {results.newAccountCount} new</span>
                    {results.continuedAccountCount > 0 && (
                      <span style={{ color: THEME.amber }}> + {results.continuedAccountCount} continued</span>
                    )}
                  </p>
                </div>
                <div style={{ display: 'flex', gap: '12px' }}>
                  <button onClick={() => { setStep('ready'); setResults(null); }} style={{ background: 'transparent', border: `1px solid ${THEME.primaryBorder}`, color: THEME.bodyOnDark, padding: '8px 16px', borderRadius: '8px', cursor: 'pointer' }}>🔄 Re-analyze</button>
                  <button onClick={() => { setStep('select-user'); setSelectedSAE(null); setAccounts([]); setResults(null); }} style={{ background: 'transparent', border: `1px solid ${THEME.primaryBorder}`, color: THEME.bodyOnDark, padding: '8px 16px', borderRadius: '8px', cursor: 'pointer' }}>← Change</button>
                </div>
              </div>
              {results.marketContext && (
                <div style={{ marginTop: '16px', padding: '16px', background: THEME.bgDark, borderRadius: '12px', borderLeft: `4px solid ${THEME.primary}` }}>
                  <div style={{ color: THEME.bodyOnDark, fontSize: '12px', marginBottom: '4px' }}>📊 Market context</div>
                  <div style={{ color: THEME.headlineOnDark, fontSize: '14px', lineHeight: 1.5 }}>{results.marketContext}</div>
                </div>
              )}
              {!storageEnabled && (
                <div style={{ marginTop: '12px', padding: '12px', background: THEME.amberMuted, borderRadius: '8px' }}>
                  <span style={{ color: THEME.amber, fontSize: '13px' }}>⚠️ Storage unavailable — rotation won't persist to next week</span>
                </div>
              )}
            </div>

            <div style={{ display: 'grid', gap: '16px' }}>
              {sortedResults.map((account, idx) => (
                <div key={idx} style={{ 
                  background: THEME.bgCard, 
                  borderRadius: '16px', 
                  border: account.isContinued 
                    ? `2px solid ${THEME.amber}` 
                    : account.isFocusAccount 
                      ? `2px solid ${THEME.focus}` 
                      : `1px solid ${THEME.cardBorder}`, 
                  overflow: 'hidden' 
                }}>
                  <div style={{ padding: '20px', cursor: 'pointer', display: 'flex', justifyContent: 'space-between', alignItems: 'center' }} onClick={() => setExpandedCard(expandedCard === idx ? null : idx)}>
                    <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
                      <div style={{ background: account.isContinued ? THEME.amber : getScoreColor(account.rank, 10), padding: '8px 16px', borderRadius: '8px', textAlign: 'center' }}>
                        <div style={{ color: THEME.headlineOnDark, fontSize: '24px', fontWeight: 700 }}>#{account.rank}</div>
                      </div>
                      <div>
                        <div style={{ display: 'flex', alignItems: 'center', gap: '8px', flexWrap: 'wrap' }}>
                          <span style={{ color: THEME.headlineOnDark, fontWeight: 600, fontSize: '18px' }}>{account.company}</span>
                          {account.isContinued ? (
                            <span style={{ background: THEME.amberMuted, color: THEME.amber, padding: '2px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 600 }}>CONTINUED</span>
                          ) : account.isFocusAccount ? (
                            <span style={{ background: THEME.focusMuted, color: THEME.focus, padding: '2px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 600 }}>FOCUS</span>
                          ) : (
                            <span style={{ background: THEME.targetMuted, color: THEME.target, padding: '2px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 600 }}>TARGET</span>
                          )}
                        </div>
                        <div style={{ color: THEME.bodyOnDark, fontSize: '14px', marginTop: '2px' }}>
                          {account.isContinued && account.continuedReason 
                            ? `Still priority: ${account.continuedReason}` 
                            : account.summary}
                        </div>
                      </div>
                    </div>
                    <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
                      <div style={{ background: THEME.bgDark, padding: '8px 16px', borderRadius: '8px', textAlign: 'center' }}>
                        <div style={{ color: account.isContinued ? THEME.amber : getScoreColor(account.rank, 10), fontSize: '24px', fontWeight: 700 }}>
                          {(account.totalScore ?? calculateWeightedScore(account.scores)).toFixed(1)}
                        </div>
                        <div style={{ color: THEME.bodyOnDark, fontSize: '11px' }}>score</div>
                      </div>
                      <div style={{ color: THEME.bodyOnDark, transform: expandedCard === idx ? 'rotate(180deg)' : 'rotate(0)', transition: 'transform 0.2s' }}>▼</div>
                    </div>
                  </div>

                  {expandedCard === idx && (
                    <div style={{ padding: '0 20px 20px', borderTop: `1px solid ${THEME.cardBorder}` }}>
                      <div style={{ marginTop: '16px' }}>
                        <div style={{ color: THEME.bodyOnDark, fontSize: '12px', marginBottom: '8px' }}>Score breakdown</div>
                        <div style={{ display: 'flex', gap: '8px', flexWrap: 'wrap' }}>
                          {CRITERIA.map(c => (
                            <div key={c.id} style={{ background: THEME.bgDark, padding: '8px 12px', borderRadius: '8px', display: 'flex', alignItems: 'center', gap: '6px' }}>
                              <span>{c.icon}</span>
                              <span style={{ color: THEME.bodyOnDark, fontSize: '12px' }}>{c.label}:</span>
                              <span style={{ color: THEME.primary, fontWeight: 600 }}>{account.scores?.[c.id] ?? 'N/A'}</span>
                            </div>
                          ))}
                        </div>
                      </div>
                      <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fit, minmax(250px, 1fr))', gap: '16px', marginTop: '16px' }}>
                        {account.currentCICD && (
                          <div style={{ background: THEME.bgDark, padding: '12px', borderRadius: '8px' }}>
                            <div style={{ color: THEME.bodyOnDark, fontSize: '11px', marginBottom: '4px' }}>🔧 Current CI/CD</div>
                            <div style={{ color: THEME.headlineOnDark, fontSize: '14px' }}>{account.currentCICD}</div>
                          </div>
                        )}
                        {account.keyTriggers?.length > 0 && (
                          <div style={{ background: THEME.bgDark, padding: '12px', borderRadius: '8px' }}>
                            <div style={{ color: THEME.bodyOnDark, fontSize: '11px', marginBottom: '4px' }}>🎯 Key triggers</div>
                            <div style={{ color: THEME.headlineOnDark, fontSize: '14px' }}>{account.keyTriggers.join(' • ')}</div>
                          </div>
                        )}
                        {account.targetPersonas?.length > 0 && (
                          <div style={{ background: THEME.bgDark, padding: '12px', borderRadius: '8px' }}>
                            <div style={{ color: THEME.bodyOnDark, fontSize: '11px', marginBottom: '4px' }}>👤 Target personas</div>
                            <div style={{ color: THEME.headlineOnDark, fontSize: '14px' }}>{account.targetPersonas.slice(0, 5).join(', ')}</div>
                          </div>
                        )}
                        {account.outreachAngle && (
                          <div style={{ background: THEME.bgDark, padding: '12px', borderRadius: '8px', gridColumn: 'span 2' }}>
                            <div style={{ color: THEME.bodyOnDark, fontSize: '11px', marginBottom: '4px' }}>💬 Outreach angle</div>
                            <div style={{ color: THEME.headlineOnDark, fontSize: '14px', fontStyle: 'italic' }}>"{account.outreachAngle}"</div>
                          </div>
                        )}
                      </div>
                      {account.sources?.length > 0 && (
                        <div style={{ marginTop: '16px' }}>
                          <div style={{ color: THEME.bodyOnDark, fontSize: '11px', marginBottom: '8px' }}>📚 Sources</div>
                          <div style={{ display: 'flex', gap: '8px', flexWrap: 'wrap' }}>
                            {account.sources.map((src, i) => (
                              <a key={i} href={src.url} target="_blank" rel="noopener noreferrer" style={{ color: THEME.primary, fontSize: '13px', textDecoration: 'none', background: THEME.primaryMuted, padding: '4px 10px', borderRadius: '4px' }}>{src.title || 'Source'}</a>
                            ))}
                          </div>
                        </div>
                      )}
                    </div>
                  )}
                </div>
              ))}
            </div>
          </div>
        )}
      </div>

      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Space+Grotesk:wght@500;600;700&display=swap');
        * { font-family: 'Inter', -apple-system, sans-serif; box-sizing: border-box; }
        @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }
        button:hover { opacity: 0.9; }
        button:active { transform: scale(0.98); }
      `}</style>
    </div>
  );
}
```

## Storage Keys

| Key Pattern | Purpose |
|-------------|---------|
| `sae-priorities:{ownerId}` | Stores last week's Top 10 account IDs |

## Weekly Rotation Flow

```
Week 1: No previous data → Show Top 10 → Save IDs
Week 2: Load Week 1 → Filter out those 10 → Show NEW Top 10 → If old accounts still score high, show as #11+ with "CONTINUED" badge → Save new IDs
Week 3: Load Week 2 → Repeat...
```

## Visual Indicators

| Badge | Color | Meaning |
|-------|-------|---------|
| FOCUS | Red (#FC5656) | Big Bet account |
| TARGET | Blue (#0057FF) | Standard target account |
| CONTINUED | Amber (#F59E0B) | Was in last week's Top 10, still high priority |
