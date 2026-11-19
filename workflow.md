# AI KYC / Onboarding Compliance Squad: 5 Agents (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `kyc-squad-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI KYC / Onboarding Compliance Squad (5 agents, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## KYC Squad (n8n Cloud) - 5 agents, 3 model providers, SYNTHETIC DATA ONLY\nRouter: Groq llama-3.1-8b (fast triage)\nDocument Verifier: Gemini (careful comparison)\nWatchlist Screener: Groq llama-3.3-70b (fuzzy name matching)\nRisk Scorer: OpenRouter free model (independent second opinion)\nDecision Writer: Gemini (clear wording)\nSetup: credentials Groq, OpenRouter, Gemini, Slack, Postgres (Supabase), SMTP. Tables kyc_watchlist (seed rows) and kyc_decisions. Slack channel kyc-reviews.\nRules in code always override the AI: watchlist hit, expired documents or high risk can never be auto-approved. Email goes to YOUR_EMAIL (demo); use {{ $json.email }} for real use.",
        "height": 340,
        "width": 640
      },
      "id": "7fbb4007-dc5b-49da-93f5-48e8b5bffeb0",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -300
      ]
    },
    {
      "parameters": {},
      "id": "cc6cdae1-19cb-4dda-bb8c-3a1f4c57ecd8",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { application_id: 'KYC-1001', name: 'Aisha Rahman', document_name: 'Aisha Rahman', document_type: 'Passport', document_expiry: '2029-05-01', country: 'BD', occupation: 'Software engineer', source_of_funds: 'Salary', monthly_volume: 3000, email: 'aisha.demo@example.com' } },\n  { json: { application_id: 'KYC-1002', name: 'Victor Petrov', document_name: 'Viktor Petrow', document_type: 'National ID', document_expiry: '2025-12-01', country: 'CY', occupation: 'Import and export trader', source_of_funds: 'Cash business', monthly_volume: 85000, email: 'victor.demo@example.com' } },\n  { json: { application_id: 'KYC-1003', name: 'Boris Fakenov', document_name: 'Boris Fakenov', document_type: 'Passport', document_expiry: '2030-01-15', country: 'IR', occupation: 'Consultant', source_of_funds: 'Gifts from relatives', monthly_volume: 40000, email: 'boris.demo@example.com' } }\n];"
      },
      "id": "c9db5a43-0282-49c2-b8eb-cf9ea4e83485",
      "name": "Mock Applications",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "operation": "select",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "kyc_watchlist",
          "mode": "name"
        },
        "options": {},
        "returnAll": true
      },
      "id": "d4599852-c144-4d96-8b43-a3c5dcf70a45",
      "name": "Get Watchlist",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        440,
        200
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      },
      "executeOnce": true
    },
    {
      "parameters": {
        "jsCode": "const list = $input.all().map((r) => r.json);\nconst text = list.map((x) => x.name + ' (' + x.country + '): ' + x.reason).join('\\n') || 'empty list';\nconst norm = (s) => String(s).toLowerCase().replace(/[^a-z]/g, '');\nreturn $('Mock Applications').all().map((a) => {\n  const hit = list.find((x) => norm(x.name) === norm(a.json.name));\n  return { json: Object.assign({}, a.json, { watchlist_text: text, watch_hit: hit ? 'yes' : 'no', watch_reason: hit ? hit.reason : '' }) };\n});"
      },
      "id": "8ea4d3e1-d29d-4d6b-a12e-fc900be801e4",
      "name": "Attach Watchlist",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        660,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Decide how deep the KYC review must be.\nApplicant country: {{ $json.country }}\nOccupation: {{ $json.occupation }}\nSource of funds: {{ $json.source_of_funds }}\nExpected monthly volume (USD): {{ $json.monthly_volume }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a KYC routing agent. Choose standard or enhanced due diligence. Choose enhanced for cash-heavy businesses, unclear source of funds, expected volume over 20000 USD per month, or countries commonly treated as high risk. Respond with ONLY a JSON object, no markdown, no extra text: {\"review_level\":\"standard|enhanced\",\"reason\":\"one sentence\"}"
        }
      },
      "id": "c805cfb9-079e-4cb0-9472-f6e2075da27a",
      "name": "Router Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        900,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "llama-3.1-8b-instant",
        "options": {}
      },
      "id": "b820c3a7-f8e3-4952-a216-49364ec859be",
      "name": "Router Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGroq",
      "typeVersion": 1,
      "position": [
        880,
        440
      ],
      "credentials": {
        "groqApi": {
          "id": "REPLACE_GROQ",
          "name": "Groq account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Attach Watchlist').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst lvl = (['standard', 'enhanced'].includes(String(r.review_level).toLowerCase()) ? String(r.review_level).toLowerCase() : 'enhanced');\nreturn { json: Object.assign({}, ctx, { review_level: lvl, routing_reason: String(r.reason || '').slice(0, 200) }) };"
      },
      "id": "f1a0c56f-ba44-424f-a079-2f06ac7a4aff",
      "name": "Parse Router",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1150,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Today's date: {{ $now.toFormat('yyyy-MM-dd') }}\nApplicant name: {{ $json.name }}\nName on the {{ $json.document_type }}: {{ $json.document_name }}\nDocument expiry: {{ $json.document_expiry }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a document verification agent. Check that the name on the document matches the applicant name (flag even small spelling differences) and that the document has not expired. Respond with ONLY a JSON object, no markdown, no extra text: {\"documents_ok\":true,\"issues\":\"none or a short list\"}"
        }
      },
      "id": "8c417e3d-4912-4910-82d1-f606a9377225",
      "name": "Document Verifier Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1420,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "6389f986-0fab-45cd-aaa2-3831dc9ec5c1",
      "name": "Document Verifier Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        1400,
        440
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Router').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst today = new Date().toISOString().slice(0, 10);\nlet ok = r.documents_ok === true || String(r.documents_ok).toLowerCase() === 'true';\nlet issues = String(r.issues || 'none');\nif (ctx.document_expiry < today) { ok = false; issues = (issues === 'none' ? '' : issues + '; ') + 'document expired'; }\nreturn { json: Object.assign({}, ctx, { documents_ok: ok ? 'yes' : 'no', doc_issues: issues.slice(0, 300) }) };"
      },
      "id": "987402cc-3779-434c-b17f-ea189ff4bce6",
      "name": "Parse Docs",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1670,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Screen this applicant against the watchlist.\nApplicant name: {{ $json.name }}\nCountry: {{ $json.country }}\n\nWatchlist:\n{{ $json.watchlist_text }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a sanctions and watchlist screening agent. Look for exact or close name matches (spelling variants, transliterations) and say how confident you are. Respond with ONLY a JSON object, no markdown, no extra text: {\"possible_match\":true,\"matched_entry\":\"name or empty\",\"confidence\":0,\"reason\":\"one sentence\"}"
        }
      },
      "id": "5f00636a-1e4c-46ec-a9a9-4316755eb3e1",
      "name": "Watchlist Screener Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1940,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {}
      },
      "id": "fb05b3d9-3f3a-4b4a-9de9-bb278a150356",
      "name": "Watchlist Screener Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGroq",
      "typeVersion": 1,
      "position": [
        1920,
        440
      ],
      "credentials": {
        "groqApi": {
          "id": "REPLACE_GROQ",
          "name": "Groq account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Docs').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst possible = r.possible_match === true || String(r.possible_match).toLowerCase() === 'true' || ctx.watch_hit === 'yes';\nreturn { json: Object.assign({}, ctx, { watch_match: possible ? 'yes' : 'no', watch_note: String(r.reason || (ctx.watch_hit === 'yes' ? 'exact watchlist match' : 'no match')).slice(0, 250) }) };"
      },
      "id": "71d3de49-1c52-4d65-a3f2-b8639ebec7be",
      "name": "Parse Screening",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2190,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Score the overall onboarding risk.\nCountry: {{ $json.country }}\nOccupation: {{ $json.occupation }}\nSource of funds: {{ $json.source_of_funds }}\nMonthly volume (USD): {{ $json.monthly_volume }}\nReview level: {{ $json.review_level }}\nDocuments OK: {{ $json.documents_ok }} ({{ $json.doc_issues }})\nWatchlist match: {{ $json.watch_match }} ({{ $json.watch_note }})",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are an onboarding risk analyst. Respond with ONLY a JSON object, no markdown, no extra text: {\"risk_level\":\"low|medium|high\",\"risk_score\":0,\"factors\":\"short list in one string\"}. risk_score is a number from 0 to 100."
        }
      },
      "id": "7bd14566-6e56-4508-9f75-eafcf438a61d",
      "name": "Risk Scorer Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        2460,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "model": "meta-llama/llama-3.3-70b-instruct:free",
        "options": {}
      },
      "id": "f8d86daa-89af-434d-b30b-e768a0121862",
      "name": "Risk Scorer Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
      "typeVersion": 1,
      "position": [
        2440,
        440
      ],
      "credentials": {
        "openRouterApi": {
          "id": "REPLACE_OPENROUTER",
          "name": "OpenRouter account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Screening').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst lvl = (['low', 'medium', 'high'].includes(String(r.risk_level).toLowerCase()) ? String(r.risk_level).toLowerCase() : 'high');\nreturn { json: Object.assign({}, ctx, { risk_level: lvl, risk_score: Math.max(0, Math.min(100, Math.round(Number(r.risk_score) || 0))), risk_factors: String(r.factors || '').slice(0, 300) }) };"
      },
      "id": "554120ca-f977-40b2-825d-7d8a75d1b024",
      "name": "Parse Risk",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2710,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Write the onboarding decision for this applicant.\nApplicant: {{ $json.name }} ({{ $json.application_id }})\nRisk level: {{ $json.risk_level }} (score {{ $json.risk_score }})\nDocuments OK: {{ $json.documents_ok }} ({{ $json.doc_issues }})\nWatchlist match: {{ $json.watch_match }} ({{ $json.watch_note }})\nRisk factors: {{ $json.risk_factors }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a compliance decision writer. Choose approve, manual_review or reject and explain briefly. Only approve when documents are fine, there is no watchlist match and risk is low. Respond with ONLY a JSON object, no markdown, no extra text: {\"decision\":\"approve|manual_review|reject\",\"rationale\":\"two sentences\",\"customer_message\":\"short polite message to the applicant that does not reveal screening details\"}"
        }
      },
      "id": "636ce65c-2fe9-4ad2-8d25-cf2582f0ffb8",
      "name": "Decision Writer Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        2980,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "5009cb79-f5ff-4ea5-b41e-a150c24fdc81",
      "name": "Decision Writer Agent Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        2960,
        440
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Parse Risk').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nlet decision = ['approve', 'manual_review', 'reject'].includes(r.decision) ? r.decision : 'manual_review';\nconst notes = [];\nif (ctx.watch_hit === 'yes') { decision = String(ctx.watch_reason).toLowerCase().includes('sanctions') ? 'reject' : 'manual_review'; notes.push('watchlist rule'); }\nelse if (ctx.watch_match === 'yes' && decision === 'approve') { decision = 'manual_review'; notes.push('possible watchlist match'); }\nif (ctx.documents_ok === 'no' && decision === 'approve') { decision = 'manual_review'; notes.push('document issue'); }\nif (ctx.risk_level === 'high' && decision === 'approve') { decision = 'manual_review'; notes.push('high risk'); }\nif (ctx.review_level === 'enhanced' && ctx.risk_level !== 'low' && decision === 'approve') { decision = 'manual_review'; notes.push('enhanced review required'); }\nreturn { json: {\n  application_id: ctx.application_id,\n  name: ctx.name,\n  country: ctx.country,\n  email: ctx.email,\n  review_level: ctx.review_level,\n  documents_ok: ctx.documents_ok,\n  watch_match: ctx.watch_match,\n  risk_level: ctx.risk_level,\n  risk_score: ctx.risk_score,\n  decision,\n  rationale: (String(r.rationale || '') + (notes.length ? ' [rules: ' + notes.join(', ') + ']' : '')).slice(0, 500),\n  customer_message: String(r.customer_message || 'Thank you for your application. We will contact you with the next steps.').slice(0, 500)\n}};"
      },
      "id": "1341012a-3173-4455-a637-1916c3ef1c96",
      "name": "Parse Decision",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        3230,
        200
      ]
    },
    {
      "parameters": {
        "rules": {
          "values": [
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "id": "0821ea04-26c9-4ec9-acac-fa233743109e",
                    "leftValue": "={{ $json.decision }}",
                    "rightValue": "approve",
                    "operator": {
                      "type": "string",
                      "operation": "equals"
                    }
                  }
                ],
                "combinator": "and"
              },
              "renameOutput": true,
              "outputKey": "approve"
            },
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "id": "1bab44f9-4408-457d-b555-c837433b886b",
                    "leftValue": "={{ $json.decision }}",
                    "rightValue": "manual_review",
                    "operator": {
                      "type": "string",
                      "operation": "equals"
                    }
                  }
                ],
                "combinator": "and"
              },
              "renameOutput": true,
              "outputKey": "manual_review"
            },
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "id": "522e57f3-8ff5-4367-9013-488039c34758",
                    "leftValue": "={{ $json.decision }}",
                    "rightValue": "reject",
                    "operator": {
                      "type": "string",
                      "operation": "equals"
                    }
                  }
                ],
                "combinator": "and"
              },
              "renameOutput": true,
              "outputKey": "reject"
            }
          ]
        },
        "options": {}
      },
      "id": "511cd178-05d0-4a43-8b13-395b3ea66e61",
      "name": "Decision?",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 3,
      "position": [
        3400,
        200
      ]
    },
    {
      "parameters": {
        "fromEmail": "YOUR_EMAIL@gmail.com",
        "toEmail": "YOUR_EMAIL@gmail.com",
        "subject": "=Your application {{ $json.application_id }}",
        "emailFormat": "text",
        "text": "=Dear {{ $json.name }},\n\n{{ $json.customer_message }}\n\nApplication: {{ $json.application_id }}",
        "options": {}
      },
      "id": "ab01964b-b23c-42ac-a12e-31b04df94ace",
      "name": "Send Approval Email",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [
        3700,
        40
      ],
      "credentials": {
        "smtp": {
          "id": "REPLACE_SMTP",
          "name": "SMTP account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "kyc-reviews",
          "mode": "name"
        },
        "text": "=:mag: *KYC {{ $json.decision }}: {{ $json.application_id }}*\nApplicant: {{ $json.name }} ({{ $json.country }})\nRisk: {{ $json.risk_level }} ({{ $json.risk_score }}) | Review: {{ $json.review_level }}\nWhy: {{ $json.rationale }}",
        "otherOptions": {}
      },
      "id": "a05168e8-b044-4f72-9afb-6bcfd41f5a48",
      "name": "Slack Review Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        3700,
        240
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "kyc_decisions",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "application_id": "={{ $json.application_id }}",
            "name": "={{ $json.name }}",
            "country": "={{ $json.country }}",
            "review_level": "={{ $json.review_level }}",
            "documents_ok": "={{ $json.documents_ok }}",
            "watch_match": "={{ $json.watch_match }}",
            "risk_level": "={{ $json.risk_level }}",
            "risk_score": "={{ $json.risk_score }}",
            "decision": "={{ $json.decision }}",
            "rationale": "={{ $json.rationale }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "application_id",
              "displayName": "application_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "name",
              "displayName": "name",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "country",
              "displayName": "country",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "review_level",
              "displayName": "review_level",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "documents_ok",
              "displayName": "documents_ok",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "watch_match",
              "displayName": "watch_match",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_level",
              "displayName": "risk_level",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_score",
              "displayName": "risk_score",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "decision",
              "displayName": "decision",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "rationale",
              "displayName": "rationale",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "37a4b5c2-0abf-480e-9683-8da6f1fe68c9",
      "name": "Save Decision",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        3700,
        440
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "Router Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Router Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Router Agent": {
      "main": [
        [
          {
            "node": "Parse Router",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Document Verifier Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Document Verifier Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Document Verifier Agent": {
      "main": [
        [
          {
            "node": "Parse Docs",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Watchlist Screener Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Watchlist Screener Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Watchlist Screener Agent": {
      "main": [
        [
          {
            "node": "Parse Screening",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Risk Scorer Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Risk Scorer Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Risk Scorer Agent": {
      "main": [
        [
          {
            "node": "Parse Risk",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Decision Writer Agent Model": {
      "ai_languageModel": [
        [
          {
            "node": "Decision Writer Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Decision Writer Agent": {
      "main": [
        [
          {
            "node": "Parse Decision",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Applications",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Applications": {
      "main": [
        [
          {
            "node": "Get Watchlist",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Watchlist": {
      "main": [
        [
          {
            "node": "Attach Watchlist",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Attach Watchlist": {
      "main": [
        [
          {
            "node": "Router Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Router": {
      "main": [
        [
          {
            "node": "Document Verifier Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Docs": {
      "main": [
        [
          {
            "node": "Watchlist Screener Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Screening": {
      "main": [
        [
          {
            "node": "Risk Scorer Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Risk": {
      "main": [
        [
          {
            "node": "Decision Writer Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Decision": {
      "main": [
        [
          {
            "node": "Decision?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Decision?": {
      "main": [
        [
          {
            "node": "Send Approval Email",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Decision",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Slack Review Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Decision",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Slack Review Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Decision",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
