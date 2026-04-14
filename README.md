
ConvoWork — Hackathon Master Plan
Absence Case Management AI · 3 People · 12 Hours · Slides + Live Demo


One wrong decision costs you 3 hours. Read this entire file before touching a keyboard.


0. The Idea in One Paragraph
ConvoWork replaces a complex absence management system's 100+ UI components with a single intelligent chat interface powered by multiple AI agents. Employees apply for leave, check balances, and upload documents by talking. Admins review cases, approve in bulk, and receive proactive alerts — all inside one conversation. The interface only loads what the user asks for (server-streamed React components), making the initial bundle roughly 10x smaller than the traditional UI. This is not a chatbot bolted onto an existing product. This is conversation as the operating system.
Tagline: Don't navigate. Just ask.

1. Team Roles — Hardcoded, No Overlap
PersonRoleOwnsPerson ALead + Agent Orchestrationapp/api/chat/, lib/agents/, agent routing logic, wiring MCP tools to ClaudePerson BMCP Servers + Datalib/store.ts, data/seed.ts, all MCP tool handlers, mock dataPerson CFrontend + Slidescomponents/, app/page.tsx, chat UI, inline UI cards, all slides
Hard rule: If you do not own a file, you do not touch it. A merge conflict at hour 9 will end you.

2. Tech Stack — Every Decision Justified
Install Commands — Run These Exactly
bashnpx create-next-app@latest convowork --typescript --tailwind --app --src-dir --no --yes

cd convowork

npm install \
  ai \
  @ai-sdk/anthropic \
  @anthropic-ai/sdk \
  @modelcontextprotocol/sdk \
  zod \
  recharts \
  react-dropzone \
  date-fns

npx shadcn@latest init
npx shadcn@latest add card badge button table scroll-area separator
Why Each Package
PackageWhy it is hereRisk if you skip itai (Vercel AI SDK v4)useChat hook + streamText + tool-calling loop built in. Handles streaming, retries, and the agent tool loop automatically. Building this from scratch takes 4 hours.You waste half the day@ai-sdk/anthropicAnthropic provider adapter for Vercel AI SDK. Required alongside raw SDK.streamText cannot talk to Claude@anthropic-ai/sdkRaw Claude API. Used directly for the document vision call (haiku model).Document AI WOW moment breaks@modelcontextprotocol/sdkOfficial MCP SDK. Makes each tool group a proper MCP server. Gives you the multi-agent architecture story that judges will love.Demo has no multi-agent narrativezodSchema validation for every MCP tool input and output. Claude needs typed schemas to call tools reliably. Also auto-generates tool descriptions.Tools hallucinate wrong parametersrechartsReact-native charting. A bar chart renders inside the chat bubble as a live component. No config hell.Balance chart WOW moment is gonereact-dropzoneDrag-and-drop file upload embedded inside the chat input. Required for the document AI moment.Your #1 WOW moment is gonedate-fnsParses "next Monday to Wednesday" into real JS Date objects for leave submission logic.Date handling breaks under demo pressureshadcn/uiProduction-quality Card, Table, Badge components in minutes. No CSS fighting.2 hours wasted on styling
Environment Variables — Create .env.local First Thing
envANTHROPIC_API_KEY=your_key_here
NEXT_PUBLIC_APP_NAME=ConvoWork
Verify the API key works before writing a single component. Run this test:
typescript// test-api.ts — run with: npx tsx test-api.ts
import Anthropic from '@anthropic-ai/sdk'
const client = new Anthropic()
const msg = await client.messages.create({
  model: 'claude-haiku-4-5-20251001',
  max_tokens: 10,
  messages: [{ role: 'user', content: 'say ok' }]
})
console.log(msg.content) // Must print before you continue
If this fails, nothing works. Fix it before Hour 2.

3. File Structure — Create This Before Writing Code
convowork/
├── .env.local
├── app/
│   ├── page.tsx                        # Main chat page (Person C)
│   ├── globals.css
│   └── api/
│       ├── chat/
│       │   └── route.ts                # Orchestrator — single POST endpoint (Person A)
│       └── set-user/
│           └── route.ts                # Switch current user for demo (Person B)
├── components/
│   ├── chat/
│   │   ├── ChatWindow.tsx              # Chat container + message list (Person C)
│   │   ├── MessageBubble.tsx           # Renders text OR inline UI card (Person C)
│   │   ├── ChatInput.tsx               # Input + file upload dropzone (Person C)
│   │   ├── UserToggle.tsx              # Switch Employee / Admin (Person C)
│   │   └── AgentTrace.tsx              # Shows which tools fired per response (Person C)
│   └── cards/
│       ├── CaseTable.tsx               # Interactive case list table (Person C)
│       ├── CaseCard.tsx                # Single case detail card (Person C)
│       ├── BalanceChart.tsx            # Recharts bar chart for leave balances (Person C)
│       └── ConfirmCard.tsx             # "Approve 4 cases? Yes / No" card (Person C)
├── lib/
│   ├── store.ts                        # In-memory DB — the entire data layer (Person B)
│   └── mcp/
│       ├── cases-tools.ts              # Case management tool definitions (Person B)
│       ├── leave-tools.ts              # Leave balance + submit tools (Person B)
│       ├── document-tools.ts           # Document extract + verify tools (Person B)
│       └── policy-tools.ts             # Policy lookup tools — build only if time (Person B)
└── data/
    └── seed.ts                         # 47 cases, 5 users, balances (Person B)

4. The Data Layer — lib/store.ts
Why In-Memory and Not JSON Files

fs.readFile is async and has aggressive caching in Next.js App Router — mutations may not reflect immediately
In-memory is synchronous, instant, zero config
Server restart resets data — completely acceptable for a 5-minute demo
Employee submits leave → instantly visible in admin's case list. That is your live mutation magic moment.

Types
typescript// lib/store.ts

export type Role = 'employee' | 'admin'

export interface User {
  id: string
  name: string
  role: Role
  department: string
  tenureYears: number
  balances?: { pto: number; sick: number; personal: number }
}

export interface LeaveCase {
  id: string
  employeeId: string
  employeeName: string
  leaveType: 'PTO' | 'Sick' | 'FMLA' | 'Maternity' | 'Paternity' | 'Bereavement' | 'Personal' | 'Intermittent'
  startDate: string
  endDate: string
  days: number
  status: 'open' | 'pending_docs' | 'approved' | 'rejected' | 'cancelled'
  priority: 'high' | 'medium' | 'low'
  docStatus: 'uploaded' | 'missing' | 'not_required'
  notes: string[]
  createdAt: string
  fmlaExpiry?: string
  rejectionReason?: string
}
Mock Users — Hardcode These Exactly
typescriptconst users: User[] = [
  {
    id: 'EMP-001', name: 'Sarah Kim', role: 'employee',
    department: 'Engineering', tenureYears: 3,
    balances: { pto: 12, sick: 5, personal: 3 }
  },
  {
    id: 'EMP-002', name: 'Marcus Reid', role: 'employee',
    department: 'Design', tenureYears: 5,
    balances: { pto: 4, sick: 8, personal: 1 }
  },
  {
    id: 'EMP-003', name: 'Priya Lal', role: 'employee',
    department: 'Analytics', tenureYears: 2,
    balances: { pto: 0, sick: 2, personal: 0 }
  },
  {
    id: 'ADMIN-001', name: 'Janet Torres', role: 'admin',
    department: 'Human Resources', tenureYears: 8
  },
  {
    id: 'ADMIN-002', name: 'David Brown', role: 'admin',
    department: 'Benefits', tenureYears: 6
  }
]

let currentUserId = 'EMP-001'
export const setCurrentUser = (id: string) => { currentUserId = id }
export const getCurrentUser = () => users.find(u => u.id === currentUserId)!
export const getUserById = (id: string) => users.find(u => u.id === id)
Mock Case Seed Distribution — 47 Total
StatusCountWhat to includeopen12Mix of all leave types, varied employees, mix of prioritiespending_docs8At least 4 FMLA, all docStatus: 'missing', some past SLAapproved18Past dates, clean — shows history depthrejected6Include rejectionReason in notescancelled3Employee-initiated
These 3 specific cases are required for the demo script — build them exactly:
typescript// Case used in Document Upload demo
{
  id: '#2341', employeeId: 'EMP-001', employeeName: 'Sarah Kim',
  leaveType: 'Sick', startDate: '2025-04-21', endDate: '2025-04-25',
  days: 5, status: 'pending_docs', priority: 'high',
  docStatus: 'missing', notes: ['Medical certificate required'],
  createdAt: new Date().toISOString()
}

// Case used in Proactive FMLA Flag demo — fmlaExpiry is 3 days from today
{
  id: '#2340', employeeId: 'EMP-002', employeeName: 'Marcus Reid',
  leaveType: 'Intermittent', startDate: '2025-01-01', endDate: '2025-12-31',
  days: 0, status: 'open', priority: 'high', docStatus: 'uploaded',
  notes: ['Intermittent FMLA approved January 2025'],
  createdAt: '2025-01-01T00:00:00.000Z',
  fmlaExpiry: new Date(Date.now() + 3 * 24 * 60 * 60 * 1000).toISOString()
}

// Case used in Complex Filter demo
{
  id: '#2338', employeeId: 'EMP-003', employeeName: 'Tom Walsh',
  leaveType: 'FMLA', startDate: '2025-04-14', endDate: '2025-04-16',
  days: 3, status: 'open', priority: 'medium', docStatus: 'missing',
  notes: ['Medical cert requested Apr 10'],
  createdAt: '2025-04-10T00:00:00.000Z'
}
Store Mutation Functions — These Must Work Perfectly
typescriptconst cases: LeaveCase[] = [...seedCases]

export const getCases = (filters?: {
  status?: string
  leaveType?: string
  docStatus?: string
  minTenureYears?: number
  employeeId?: string
}) => {
  let result = [...cases]
  if (filters?.status) result = result.filter(c => c.status === filters.status)
  if (filters?.leaveType) result = result.filter(c => c.leaveType === filters.leaveType)
  if (filters?.docStatus) result = result.filter(c => c.docStatus === filters.docStatus)
  if (filters?.employeeId) result = result.filter(c => c.employeeId === filters.employeeId)
  if (filters?.minTenureYears) {
    result = result.filter(c => {
      const user = getUserById(c.employeeId)
      return (user?.tenureYears ?? 0) >= filters.minTenureYears!
    })
  }
  return result
}

export const getCaseById = (id: string) => cases.find(c => c.id === id)

export const submitLeave = (data: Omit<LeaveCase, 'id' | 'createdAt' | 'notes' | 'status'>) => {
  const newCase: LeaveCase = {
    ...data,
    id: `#${2348 + cases.length}`,
    status: data.docStatus === 'not_required' ? 'open' : 'pending_docs',
    notes: ['Submitted via ConvoWork AI'],
    createdAt: new Date().toISOString()
  }
  cases.unshift(newCase) // Top of admin list — live demo magic
  return newCase
}

export const approveCase = (id: string) => {
  const c = cases.find(x => x.id === id)
  if (c) { c.status = 'approved'; c.notes.push(`Approved ${new Date().toLocaleDateString()}`) }
  return c
}

export const bulkApprove = (ids: string[]) =>
  ids.map(id => approveCase(id)).filter(Boolean)

export const rejectCase = (id: string, reason: string) => {
  const c = cases.find(x => x.id === id)
  if (c) { c.status = 'rejected'; c.rejectionReason = reason; c.notes.push(`Rejected: ${reason}`) }
  return c
}

export const addNote = (id: string, note: string) => {
  const c = cases.find(x => x.id === id)
  if (c) c.notes.push(note)
  return c
}

export const getBalance = (userId: string) => getUserById(userId)?.balances ?? null

export const deductBalance = (userId: string, leaveType: string, days: number) => {
  const user = getUserById(userId)
  if (!user?.balances) return
  const key = leaveType === 'PTO' ? 'pto' : leaveType === 'Sick' ? 'sick' : 'personal'
  ;(user.balances as any)[key] = Math.max(0, (user.balances as any)[key] - days)
}

5. MCP Tool Definitions — lib/mcp/
Wire these as tool objects passed into streamText. Define them using the Vercel AI SDK tool() helper with Zod schemas. No HTTP MCP servers needed — direct tool definitions work perfectly for the demo and are simpler.
cases-tools.ts
typescriptimport { tool } from 'ai'
import { z } from 'zod'
import * as store from '../store'

export const casesTools = {
  list_cases: tool({
    description: 'List and filter absence cases. Use for admin case review, searching, and filtering by any combination of status, leave type, document status, or employee tenure. Always use this when an admin asks to see cases.',
    parameters: z.object({
      status: z.enum(['open','pending_docs','approved','rejected','cancelled']).optional(),
      leaveType: z.string().optional(),
      docStatus: z.enum(['uploaded','missing','not_required']).optional(),
      minTenureYears: z.number().optional(),
      employeeId: z.string().optional(),
      limit: z.number().optional().default(10)
    }),
    execute: async (params) => {
      const results = store.getCases(params).slice(0, params.limit ?? 10)
      return { cases: results, total: results.length, ui_component: 'CaseTable' }
    }
  }),

  get_case: tool({
    description: 'Get full details of a specific case by ID. Also returns FMLA days remaining if applicable.',
    parameters: z.object({ id: z.string() }),
    execute: async ({ id }) => {
      const c = store.getCaseById(id)
      if (!c) return { error: `Case ${id} not found` }
      const user = store.getUserById(c.employeeId)
      const fmlaDaysRemaining = c.fmlaExpiry
        ? Math.ceil((new Date(c.fmlaExpiry).getTime() - Date.now()) / 86400000)
        : null
      return { case: c, employeeTenure: user?.tenureYears, fmlaDaysRemaining, ui_component: 'CaseCard' }
    }
  }),

  approve_case: tool({
    description: 'Approve a single leave case by ID',
    parameters: z.object({ id: z.string() }),
    execute: async ({ id }) => ({ case: store.approveCase(id), action: 'approved' })
  }),

  bulk_approve: tool({
    description: 'Approve multiple cases at once. Only call this after showing the user a confirmation list and receiving explicit confirmation like "yes" or "confirm".',
    parameters: z.object({ ids: z.array(z.string()) }),
    execute: async ({ ids }) => ({
      cases: store.bulkApprove(ids),
      count: ids.length,
      action: 'bulk_approved',
      ui_component: 'CaseTable'
    })
  }),

  reject_case: tool({
    description: 'Reject a leave case with a reason',
    parameters: z.object({ id: z.string(), reason: z.string() }),
    execute: async ({ id, reason }) => ({ case: store.rejectCase(id, reason), action: 'rejected' })
  }),

  add_note: tool({
    description: 'Add a note or comment to a case',
    parameters: z.object({ id: z.string(), note: z.string() }),
    execute: async ({ id, note }) => ({ case: store.addNote(id, note) })
  })
}
leave-tools.ts
typescriptexport const leaveTools = {
  get_balance: tool({
    description: 'Get current leave balance for an employee. Returns PTO, sick, and personal day counts. Always call this when an employee asks about their balance or before applying for leave.',
    parameters: z.object({ userId: z.string() }),
    execute: async ({ userId }) => ({
      balances: store.getBalance(userId),
      userId,
      ui_component: 'BalanceChart'
    })
  }),

  check_eligibility: tool({
    description: 'Check if an employee is eligible for a specific leave type and number of days. Always call this before submit_leave.',
    parameters: z.object({
      userId: z.string(),
      leaveType: z.string(),
      days: z.number()
    }),
    execute: async ({ userId, leaveType, days }) => {
      const balance = store.getBalance(userId)
      if (!balance) return { eligible: false, reason: 'User not found' }
      const key = leaveType === 'PTO' ? 'pto' : leaveType === 'Sick' ? 'sick' : 'personal'
      const available = (balance as any)[key] ?? 0
      return {
        eligible: available >= days,
        available,
        requested: days,
        reason: available >= days
          ? 'Sufficient balance available'
          : `Only ${available} days available, ${days} requested`
      }
    }
  }),

  submit_leave: tool({
    description: 'Submit a leave request. Only call after check_eligibility confirms the employee is eligible.',
    parameters: z.object({
      userId: z.string(),
      leaveType: z.enum(['PTO','Sick','FMLA','Maternity','Paternity','Bereavement','Personal','Intermittent']),
      startDate: z.string(),
      endDate: z.string(),
      days: z.number(),
      requiresDoc: z.boolean()
    }),
    execute: async (params) => {
      const user = store.getUserById(params.userId)
      const newCase = store.submitLeave({
        employeeId: params.userId,
        employeeName: user?.name ?? 'Unknown',
        leaveType: params.leaveType,
        startDate: params.startDate,
        endDate: params.endDate,
        days: params.days,
        priority: 'medium',
        docStatus: params.requiresDoc ? 'missing' : 'not_required'
      })
      if (!params.requiresDoc) {
        store.deductBalance(params.userId, params.leaveType, params.days)
      }
      return { case: newCase, ui_component: 'CaseCard' }
    }
  }),

  get_my_cases: tool({
    description: 'Get all leave cases for the current employee',
    parameters: z.object({ userId: z.string() }),
    execute: async ({ userId }) => ({
      cases: store.getCases({ employeeId: userId }),
      ui_component: 'CaseTable'
    })
  })
}
document-tools.ts
typescriptimport Anthropic from '@anthropic-ai/sdk'
const anthropic = new Anthropic()

export const documentTools = {
  extract_document_fields: tool({
    description: 'Extract structured data from an uploaded medical certificate image. Returns doctor name, hospital, recommended leave dates, and diagnosis type. Use when employee uploads a document.',
    parameters: z.object({ imageBase64: z.string() }),
    execute: async ({ imageBase64 }) => {
      const response = await anthropic.messages.create({
        model: 'claude-haiku-4-5-20251001', // Fast + cheap for vision extraction
        max_tokens: 400,
        messages: [{
          role: 'user',
          content: [
            {
              type: 'image',
              source: { type: 'base64', media_type: 'image/jpeg', data: imageBase64 }
            },
            {
              type: 'text',
              text: `Extract fields from this medical certificate. Return ONLY valid JSON, no other text:
{
  "doctorName": "string or null",
  "hospital": "string or null",
  "recommendedRestStart": "YYYY-MM-DD or null",
  "recommendedRestEnd": "YYYY-MM-DD or null",
  "diagnosisType": "brief general description",
  "signatureDetected": true or false
}`
            }
          ]
        }]
      })
      const raw = (response.content[0] as any).text
      try {
        return { extracted: JSON.parse(raw), success: true }
      } catch {
        return { error: 'Could not parse document', success: false }
      }
    }
  }),

  mark_document_uploaded: tool({
    description: 'Mark a leave case as having its supporting document uploaded and verified',
    parameters: z.object({ caseId: z.string() }),
    execute: async ({ caseId }) => {
      const c = store.getCaseById(caseId)
      if (c) {
        c.docStatus = 'uploaded'
        c.notes.push('Document verified by ConvoWork AI')
        if (c.status === 'pending_docs') c.status = 'open'
      }
      return { case: c }
    }
  })
}

6. The Chat API Route — Person A's Core File
typescript// app/api/chat/route.ts
import { streamText } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'
import { casesTools } from '@/lib/mcp/cases-tools'
import { leaveTools } from '@/lib/mcp/leave-tools'
import { documentTools } from '@/lib/mcp/document-tools'
import { getCurrentUser } from '@/lib/store'

export const runtime = 'nodejs' // Required — edge runtime has no Node.js APIs

export async function POST(req: Request) {
  const { messages } = await req.json()
  const user = getCurrentUser()

  const systemPrompt = `You are ConvoWork, an intelligent absence management assistant.
Current user: ${user.name} | Role: ${user.role} | Department: ${user.department} | ID: ${user.id}

BEHAVIOR RULES — follow these exactly:
1. For leave applications: always call check_eligibility first, then submit_leave
2. For bulk actions: list what you will do and ask "Shall I proceed?" — only call bulk_approve after user confirms
3. For any case view via get_case: always check fmlaDaysRemaining in the response — if it is under 7, immediately flag it to the admin even if they did not ask
4. Be concise. Maximum 2–3 sentences unless you are presenting data.
5. When tools return ui_component, do not describe the data in text — the UI card will show it. Just confirm what was done.
6. Never make up case IDs or user data. Only use what tools return.

${user.role === 'admin'
  ? `You have full access to all ${user.name}'s case queue. Be proactive — flag SLA breaches, missing documents, and FMLA expiries even when not asked.`
  : `You are helping ${user.name} (ID: ${user.id}). Always use their ID when checking balance or submitting leave. Guide them step by step.`
}`

  const result = await streamText({
    model: anthropic('claude-sonnet-4-5-20250514'),
    system: systemPrompt,
    messages,
    tools: {
      ...casesTools,
      ...leaveTools,
      ...documentTools,
    },
    maxSteps: 5,
  })

  return result.toDataStreamResponse()
}
typescript// app/api/set-user/route.ts
import { setCurrentUser } from '@/lib/store'

export async function POST(req: Request) {
  const { userId } = await req.json()
  setCurrentUser(userId)
  return Response.json({ ok: true })
}

7. Frontend Key Components — Person C
ChatWindow.tsx — Wire to useChat
typescript'use client'
import { useChat } from 'ai/react'
import { MessageBubble } from './MessageBubble'
import { ChatInput } from './ChatInput'
import { UserToggle } from './UserToggle'

export function ChatWindow() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: '/api/chat'
  })

  return (
    <div className="flex flex-col h-screen max-w-3xl mx-auto">
      <header className="flex items-center justify-between p-4 border-b">
        <h1 className="font-medium text-lg">ConvoWork</h1>
        <UserToggle />
      </header>
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map(m => <MessageBubble key={m.id} message={m} />)}
      </div>
      <ChatInput
        input={input}
        onChange={handleInputChange}
        onSubmit={handleSubmit}
        isLoading={isLoading}
      />
    </div>
  )
}
MessageBubble.tsx — The Heart of the Demo
This is where text becomes UI components. When a tool result contains ui_component, render the card.
typescript'use client'
import { CaseTable } from '@/components/cards/CaseTable'
import { BalanceChart } from '@/components/cards/BalanceChart'
import { CaseCard } from '@/components/cards/CaseCard'

const CARD_MAP: Record<string, React.ComponentType<any>> = {
  CaseTable,
  BalanceChart,
  CaseCard,
}

export function MessageBubble({ message }: { message: any }) {
  const isUser = message.role === 'user'

  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'}`}>
      <div className={`max-w-xl rounded-2xl px-4 py-3 ${isUser ? 'bg-primary text-primary-foreground' : 'bg-muted'}`}>
        {/* Render text content */}
        {message.content && typeof message.content === 'string' && (
          <p className="text-sm">{message.content}</p>
        )}

        {/* Render tool results as UI components */}
        {message.toolInvocations?.map((tool: any) => {
          if (tool.state !== 'result') return null
          const result = tool.result
          const Component = result?.ui_component ? CARD_MAP[result.ui_component] : null
          if (Component) return <Component key={tool.toolCallId} data={result} />
          return null
        })}
      </div>
    </div>
  )
}
BalanceChart.tsx — Renders Inside Chat
typescriptimport { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, Cell } from 'recharts'

export function BalanceChart({ data }: { data: { balances: any } }) {
  if (!data.balances) return null
  const chartData = [
    { name: 'PTO', days: data.balances.pto, color: '#1D9E75' },
    { name: 'Sick', days: data.balances.sick, color: '#7F77DD' },
    { name: 'Personal', days: data.balances.personal, color: '#378ADD' },
  ]
  return (
    <div className="mt-2 rounded-xl border bg-card p-4 w-72">
      <p className="text-xs font-medium text-muted-foreground mb-3">Leave balance</p>
      <ResponsiveContainer width="100%" height={100}>
        <BarChart data={chartData} barSize={28}>
          <XAxis dataKey="name" tick={{ fontSize: 11 }} axisLine={false} tickLine={false} />
          <YAxis tick={{ fontSize: 11 }} axisLine={false} tickLine={false} />
          <Tooltip cursor={false} />
          <Bar dataKey="days" radius={[4,4,0,0]}>
            {chartData.map((entry, i) => <Cell key={i} fill={entry.color} />)}
          </Bar>
        </BarChart>
      </ResponsiveContainer>
    </div>
  )
}
UserToggle.tsx — Critical for Demo
typescript'use client'
import { useState } from 'react'

const USERS = [
  { id: 'EMP-001', name: 'Sarah Kim', label: 'Employee' },
  { id: 'EMP-002', name: 'Marcus Reid', label: 'Employee' },
  { id: 'ADMIN-001', name: 'Janet Torres', label: 'Admin' },
]

export function UserToggle() {
  const [current, setCurrent] = useState(USERS[0])

  const handleSwitch = async (userId: string) => {
    const user = USERS.find(u => u.id === userId)!
    setCurrent(user)
    await fetch('/api/set-user', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId })
    })
    // Reload page to clear chat history for clean demo
    window.location.reload()
  }

  return (
    <select
      value={current.id}
      onChange={e => handleSwitch(e.target.value)}
      className="text-sm border rounded-lg px-3 py-1.5 bg-background"
    >
      {USERS.map(u => (
        <option key={u.id} value={u.id}>{u.name} ({u.label})</option>
      ))}
    </select>
  )
}

8. Claude Code Prompts — Copy-Paste Into VSCode
Use these prompts exactly. Each is self-contained and produces a working file.

Prompt 1 — data/seed.ts (Person B, Hour 1)
Create a TypeScript file at data/seed.ts that exports an array of 47 LeaveCase objects for a leave management demo app.

LeaveCase type:
{ id: string, employeeId: string, employeeName: string, leaveType: 'PTO'|'Sick'|'FMLA'|'Maternity'|'Paternity'|'Bereavement'|'Personal'|'Intermittent', startDate: string, endDate: string, days: number, status: 'open'|'pending_docs'|'approved'|'rejected'|'cancelled', priority: 'high'|'medium'|'low', docStatus: 'uploaded'|'missing'|'not_required', notes: string[], createdAt: string, fmlaExpiry?: string, rejectionReason?: string }

Distribution: 12 open, 8 pending_docs, 18 approved, 6 rejected, 3 cancelled.

Use these employee names across the cases (spread them evenly): Sarah Kim (EMP-001), Marcus Reid (EMP-002), Priya Lal (EMP-003), Tom Walsh (EMP-004), Elena Cruz (EMP-005), Nina Patel (EMP-006), James Liu (EMP-007), Raj Mehta (EMP-008), Lisa Cho (EMP-009), Ben Osei (EMP-010), Ana Torres (EMP-011), Chris Park (EMP-012).

Make dates realistic — spanning last 3 months to next 2 months. Include 1–3 realistic notes per case.

These 3 cases must be included exactly as specified:
1. { id: '#2341', employeeId: 'EMP-001', employeeName: 'Sarah Kim', leaveType: 'Sick', startDate: '2025-04-21', endDate: '2025-04-25', days: 5, status: 'pending_docs', priority: 'high', docStatus: 'missing', notes: ['Medical certificate required'], createdAt: new Date().toISOString() }
2. { id: '#2340', employeeId: 'EMP-002', employeeName: 'Marcus Reid', leaveType: 'Intermittent', startDate: '2025-01-01', endDate: '2025-12-31', days: 0, status: 'open', priority: 'high', docStatus: 'uploaded', notes: ['Intermittent FMLA approved January 2025'], createdAt: '2025-01-01T00:00:00.000Z', fmlaExpiry: new Date(Date.now() + 3 * 24 * 60 * 60 * 1000).toISOString() }
3. { id: '#2338', employeeId: 'EMP-004', employeeName: 'Tom Walsh', leaveType: 'FMLA', startDate: '2025-04-14', endDate: '2025-04-16', days: 3, status: 'open', priority: 'medium', docStatus: 'missing', notes: ['Medical cert requested Apr 10'], createdAt: '2025-04-10T00:00:00.000Z' }

Export as: export const seedCases: LeaveCase[] = [...]

Prompt 2 — lib/store.ts (Person B, Hour 2)
Create lib/store.ts — an in-memory singleton database for a leave management demo app. No external DB. All state is module-level arrays/variables.

Import LeaveCase and User types from '@/data/seed' (define them there too) and seedCases from '@/data/seed'.

Users to hardcode:
- EMP-001: Sarah Kim, employee, Engineering, tenureYears 3, balances {pto:12, sick:5, personal:3}
- EMP-002: Marcus Reid, employee, Design, tenureYears 5, balances {pto:4, sick:8, personal:1}
- EMP-003: Priya Lal, employee, Analytics, tenureYears 2, balances {pto:0, sick:2, personal:0}
- ADMIN-001: Janet Torres, admin, Human Resources, tenureYears 8
- ADMIN-002: David Brown, admin, Benefits, tenureYears 6

Export these functions:
- setCurrentUser(id: string): void
- getCurrentUser(): User
- getUserById(id: string): User | undefined
- getUsers(): User[]
- getCases(filters?: {status?, leaveType?, docStatus?, minTenureYears?, employeeId?}): LeaveCase[]
- getCaseById(id: string): LeaveCase | undefined
- submitLeave(data): LeaveCase — auto-generates id like #2349, #2350 etc, unshifts to front of array
- approveCase(id: string): LeaveCase | undefined
- bulkApprove(ids: string[]): LeaveCase[]
- rejectCase(id: string, reason: string): LeaveCase | undefined
- addNote(id: string, note: string): LeaveCase | undefined
- getBalance(userId: string): {pto,sick,personal} | null
- deductBalance(userId: string, leaveType: string, days: number): void

getCases must support filtering by minTenureYears by cross-referencing the users array tenureYears.
submitLeave sets status to 'pending_docs' if docStatus is 'missing', otherwise 'open'.

Prompt 3 — components/cards/CaseTable.tsx (Person C, Hour 4)
Create a React component at components/cards/CaseTable.tsx using shadcn/ui Table and Badge. TypeScript. 

Props: { data: { cases: any[], total: number } }

Render a clean table with these columns: Case ID, Employee Name, Leave Type, Dates (startDate to endDate), Status (Badge), Priority (Badge), Docs.

Badge colors:
- Status: open=blue, pending_docs=yellow, approved=green, rejected=red, cancelled=gray
- Priority: high=red, medium=yellow, low=green  
- Docs: uploaded=green, missing=red, not_required=gray

Show a header: "{total} cases" in muted text above the table.
Max height 300px with overflow-y scroll on the table body.
Font size 13px throughout. Compact row padding (py-2).
Tailwind + shadcn only. No other libraries.

Prompt 4 — components/cards/CaseCard.tsx (Person C, Hour 5)
Create a React component at components/cards/CaseCard.tsx using shadcn/ui Card, Badge. TypeScript.

Props: { data: { case: any, fmlaDaysRemaining?: number | null, employeeTenure?: number } }

Show a card with:
- Header: case ID (monospace) + status badge + priority badge
- Employee name, leave type, dates, days count
- Document status badge
- Notes as a bulleted list (small, muted text)
- If fmlaDaysRemaining is not null and under 7: show a prominent yellow warning banner at the top: "⚠️ FMLA expires in X days — renewal required"
- If employeeTenure: show "X years tenure" in muted text

Keep it compact. Max width 400px. Tailwind + shadcn only.

Prompt 5 — components/cards/ConfirmCard.tsx (Person C, Hour 6)
Create a React component at components/cards/ConfirmCard.tsx. TypeScript.

Props: { cases: any[], onConfirm: () => void, onCancel: () => void }

Show a card with:
- Title: "Approve {cases.length} cases?"
- A list of case IDs and employee names
- Two buttons: "Yes, approve all" (primary) and "Cancel" (outline)
- Compact, max width 360px

Note: In the actual demo, the "Yes" confirmation will be typed in the chat. This card is just for display — the buttons can call the passed onConfirm/onCancel props or be left as visual-only placeholders.

9. The 5 WOW Moments — Priority Order
Build in this order. If time runs out, cut from the bottom.
WOW #1 — Document Reads Itself (build by Hour 7 — Person A + B + C)
Employee uploads a medical certificate image → haiku model reads it → extracts dates + doctor → auto-submits leave → case appears in admin list.

Person B: extract_document_fields tool in document-tools.ts
Person C: File dropzone in ChatInput, converts to base64, sends with message
Person A: Wire imageBase64 through from frontend to the tool

Say during demo: "She did not fill a single field. The document was the form."
WOW #2 — One Sentence, Five Dropdowns (already works when list_cases works)
Admin types: "Show me all open FMLA cases from last month where documents are missing and the employee has been here more than 2 years"
CaseTable renders with exactly those results.
No extra code required beyond list_cases working correctly. Test this query in Hour 6.
Say during demo: "Five filter dropdowns in the old UI. One sentence here."
WOW #3 — Proactive FMLA Flag (already works when get_case + seed data works)
Admin asks about Marcus Reid (#2340) → fmlaDaysRemaining returns 3 → system prompt rule fires → AI flags it unprompted.
No extra code — just system prompt instruction + correct fmlaExpiry in seed. Verify in Hour 6.
Say during demo: "Nobody asked it to check that. It just knew."
WOW #4 — Bulk Approve in Chat (build by Hour 7)
Admin: "Approve all low-risk PTO cases this week" → AI lists them → Admin says "yes" → bulk_approve fires → all approved.

Person B: bulk_approve tool
Person C: ConfirmCard component (optional — text response also works)

Say during demo: "Twenty clicks in the old UI. Two words here."
WOW #5 — Agent Trace (build Hour 9–10 only if ahead of schedule)
After each response, small chips show which tools fired: check_eligibility → submit_leave → get_balance.

Read message.toolInvocations from the AI SDK message object, show tool names as badges.

Say during demo: "Three tools coordinated on that response in real time."

10. Hour-by-Hour Timeline
Hour 1 — All Together (do not split yet)

 Run the exact install commands from Section 2
 Create .env.local, run the API key test — must pass before moving on
 Create full folder structure from Section 3 (empty files, correct paths)
 Person B begins seed.ts with Claude Code Prompt 1
 Person A reads Sections 5 and 6 fully
 Person C sets up shadcn and verifies components install

Hours 2–3 — Split
Person APerson BPerson CBuild app/api/chat/route.ts (Prompt 4)Build lib/store.ts (Prompt 2)Build app/page.tsx + ChatWindow.tsx + basic useChat wiringTest: POST to /api/chat returns a streamTest: import getCases(), console.log result countTest: Typing in chat returns a text response
Hours 4–6 — Split
Person APerson BPerson CWire all MCP tools into chat routeBuild cases-tools.ts + leave-tools.tsBuild CaseTable.tsx (Prompt 3) + BalanceChart.tsxTest: "show me open cases" → tool fires + CaseTable rendersBuild document-tools.ts with vision callBuild CaseCard.tsx (Prompt 4) + UserToggle.tsxTest: "what is my balance" → BalanceChart rendersTest each tool function in isolationTest: components render with hardcoded props
Hours 6–8 — Integration
Person APerson BPerson CTest WOW #2 complex filter query end-to-endTest WOW #1 document upload with a real imageWire MessageBubble.tsx to render correct card from ui_component fieldTest WOW #3 proactive flag fires on Marcus's caseVerify submitLeave unshifts and case appears at top of admin listBuild ChatInput.tsx with file dropzoneTest WOW #4 bulk approve with confirm flowMake sure seed has exactly 47 cases with correct demo casesBuild ConfirmCard.tsx
Hours 8–9 — Bug Fix Only
Run every WOW moment in sequence. Fix only what breaks the demo script. No new features.
Hours 9–10 — Polish + Slides

Person A: Agent trace chips if ahead of schedule
Person B: Verify seed data looks impressive in the CaseTable (real names, realistic dates, varied types)
Person C: Build all 5 slides

Hours 10–11 — Full Rehearsal

Run the complete 5-minute demo script 3 times out loud
Time each section with a stopwatch
Fix only what breaks the demo

Hour 12 — Freeze

Zero new code
Verify npm run dev starts cleanly on a fresh terminal
Have the slide deck open and ready on a separate screen before walking in


11. Slide Structure — 5 Slides Maximum
SlideContentTime1 — The ProblemShow a screenshot of a complex absence UI with arrows counting: "8 clicks, 3 pages, 12 fields — to submit one leave request." Last line: "We asked: what if this was just a conversation?"30 sec2 — The IdeaTagline large: "Don't navigate. Just ask." One simple diagram: User → Chat → [Leave Agent · Case Agent · Doc Agent] → Action + Live UI. Below: "Initial bundle: 180KB — components load only when asked."45 sec3 — LIVE DEMOSwitch to browser. This is 3 minutes 30 seconds. Do not rush it.3:304 — ArchitectureThree columns: MCP Tool Groups (cases, leave, document, policy) → Claude Orchestrator (sonnet for chat, haiku for vision) → Streaming UI Components (CaseTable, BalanceChart, CaseCard). One line: "Modular — any tool plugs in without touching the chat layer."30 sec5 — ImpactThree large numbers: 90% fewer clicks · Zero form fields for document leaves · 10x smaller initial load. Final line: "Don't navigate. Just ask."15 sec

12. The 5-Minute Demo Script — Exact Words
Rehearse this until you can do it without reading.
0:00–0:30 — The problem
Open the old absence UI. Count clicks out loud. Say: "To submit one leave request — 8 clicks, three page loads, 12 form fields. Here is what we built instead."
0:30–1:30 — WOW #1: Document reads itself
Switch to Sarah Kim in the user toggle. Type: "I have a medical certificate, can I apply for sick leave?" Upload the cert image. Watch the AI extract fields and submit the case. Say: "She did not fill a single field. The document was the form."
1:30–2:00 — Balance chart in chat
Type: "What is my leave balance?" Chart renders inside the chat bubble. Say: "That chart only downloaded when she asked for it. The entire page load before this was 180 kilobytes."
2:00–3:00 — Admin: complex filter + proactive flag
Switch to Janet Torres. Type: "Show me all open FMLA cases where documents are missing and the employee has been here more than 2 years." Table renders. Say: "Five filter dropdowns in the old system. One sentence here."
Type: "Show me Marcus Reid's case." Case card renders. AI flags FMLA expiry. Say: "Nobody asked it to check that. It just knew to look."
3:00–3:45 — Bulk approve
Type: "Approve all low-risk PTO cases this week." Confirm appears. Type "yes." Cases approved. Say: "Twenty clicks. Two words."
3:45–4:15 — Architecture slide
Switch to slide 4. Say: "Under the hood — four MCP tool groups, each owning one domain. Claude routes to the right tool. Any capability plugs in without touching the chat layer."
4:15–5:00 — Close
Switch to slide 5. Say: "ConvoWork is not a chatbot bolted onto a leave system. It is the leave system — with conversation as the interface. Every action, every insight, every document. One place. Plain language. No navigation." Pause. "Don't navigate. Just ask."

13. Do Not Build These
Every hour spent here is stolen from the demo.

❌ Authentication / login / sessions
❌ Real database or file system storage
❌ API error handling beyond a basic try/catch
❌ Loading states or skeleton screens
❌ Mobile responsive layout
❌ Unit or integration tests
❌ Deployment pipeline
❌ More than 6 distinct chat interactions
❌ Any feature not referenced in the 5-minute demo script


14. Pre-Demo Verification Checklist
Run this the night before. Binary pass/fail only.

 npm run dev starts with zero console errors
 API key in .env.local returns a valid response (run the test snippet)
 UserToggle switches between Sarah Kim and Janet Torres correctly
 "What is my leave balance?" → BalanceChart renders inside the chat bubble
 Uploading a medical cert image → fields extracted → leave submitted → case #2348+ appears in admin list
 Switching to Janet Torres → complex FMLA filter query → CaseTable renders with correct filtered results
 "Show me Marcus Reid's case" → CaseCard renders → FMLA expiry warning appears without being asked
 "Approve all low-risk PTO cases" → confirmation appears → "yes" → cases flip to approved
 Sarah's newly submitted case appears at the very top of Janet's case list
 All 5 slides are finalized and the deck is ready to open on a separate screen
 You have rehearsed the 5-minute script at least 3 times with a timer


Freeze this file after Hour 1. If you are editing it at Hour 8, something has gone wrong.
