# Nexora — Frontend Web (Next.js)

## Project Context
Nexora là nền tảng cộng tác học tập cho sinh viên: Task Management (Kanban) + Study Schedule + Forum + AI Assistant.
Stack: **Next.js 16** (App Router) · React 19 · TypeScript 5 · Tailwind CSS v4 · shadcn/ui · TanStack Query v5 · Zustand v5

Docs đầy đủ: `docs/` (Nexora-Docs submodule)
- API conventions: `docs/conventions/api-design.md`
- FE setup guide: `docs/fe/01-setup.md`
- API specs (để tích hợp): `docs/be/modules/`

---

## Project Structure
```
src/
├── app/                    # Next.js App Router — pages và layouts
│   ├── (auth)/             # Route group — trang login/register (no sidebar)
│   ├── (dashboard)/        # Route group — trang chính (có sidebar)
│   │   ├── tasks/
│   │   ├── schedule/
│   │   ├── forum/
│   │   └── groups/
│   ├── layout.tsx          # Root layout
│   └── globals.css         # Tailwind v4 global styles
├── components/
│   ├── ui/                 # shadcn/ui components (không edit trực tiếp)
│   ├── common/             # Shared components (Button, Modal, Table...)
│   ├── tasks/              # Task-specific components
│   ├── schedule/           # Schedule/Calendar components
│   ├── forum/              # Forum components
│   └── layout/             # Sidebar, Navbar, Header
├── hooks/                  # Custom React hooks
├── lib/
│   ├── api.ts              # Axios instance với interceptors
│   ├── utils.ts            # cn() utility, date helpers
│   └── validations/        # Zod schemas cho forms
├── stores/                 # Zustand stores
│   ├── auth.store.ts       # User session, Access Token
│   ├── ui.store.ts         # Sidebar open/close, theme
│   └── notification.store.ts
├── types/                  # TypeScript interfaces, API response types
└── services/               # API call functions (dùng với TanStack Query)
    ├── auth.service.ts
    ├── tasks.service.ts
    └── ...
```

---

## Next.js App Router — CRITICAL Rules

### Server vs Client Components
```typescript
// ✅ DEFAULT: Server Component (không có 'use client')
// - Fetch data trực tiếp
// - Không dùng hooks (useState, useEffect)
// - Render nhanh hơn, SEO tốt hơn

// ✅ Client Component — chỉ khi CẦN interactivity
'use client'
// - Dùng hooks
// - Dùng browser APIs
// - Xử lý events
```

**Quy tắc:** Đẩy `'use client'` xuống thấp nhất có thể trong component tree.

```typescript
// ✅ Đúng — Server Component page, Client Component chỉ cho interactive part
// app/(dashboard)/tasks/page.tsx (Server Component)
export default async function TasksPage() {
  const tasks = await getTasks()  // server-side fetch
  return <TaskBoard initialTasks={tasks} />  // pass data xuống
}

// components/tasks/TaskBoard.tsx (Client Component)
'use client'
export function TaskBoard({ initialTasks }: Props) {
  const [tasks, setTasks] = useState(initialTasks)
  // drag & drop logic...
}
```

### Data Fetching
```typescript
// Server Component → fetch trực tiếp
async function getTasksFromServer() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/tasks`, {
    headers: { Authorization: `Bearer ${getServerToken()}` },
    next: { revalidate: 60 }  // ISR
  })
  return res.json()
}

// Client Component → TanStack Query
const { data, isLoading, error } = useQuery({
  queryKey: ['tasks', filters],
  queryFn: () => tasksService.getAll(filters),
})
```

---

## API Integration

### Axios Instance (`src/lib/api.ts`)
```typescript
// Dùng axiosInstance, không dùng fetch trực tiếp trong client components
import { api } from '@/lib/api'

// Interceptor tự động:
// - Attach Access Token vào header
// - Token refresh tự động khi 401
// - Retry một lần sau khi refresh thành công
```

### TanStack Query Patterns
```typescript
// Query keys phải nhất quán — dùng factory pattern
export const taskKeys = {
  all: ['tasks'] as const,
  list: (filters: FilterTaskDto) => [...taskKeys.all, 'list', filters] as const,
  detail: (id: string) => [...taskKeys.all, 'detail', id] as const,
}

// Mutation với optimistic update
const mutation = useMutation({
  mutationFn: tasksService.updateStatus,
  onMutate: async (newData) => {
    await queryClient.cancelQueries({ queryKey: taskKeys.all })
    const prev = queryClient.getQueryData(taskKeys.list(filters))
    queryClient.setQueryData(taskKeys.list(filters), optimisticUpdate(prev, newData))
    return { prev }
  },
  onError: (_, __, context) => {
    queryClient.setQueryData(taskKeys.list(filters), context?.prev)
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: taskKeys.all }),
})
```

---

## State Management (Zustand)

```typescript
// stores/auth.store.ts
interface AuthStore {
  user: UserProfile | null
  accessToken: string | null  // lưu trong memory, KHÔNG localStorage
  setUser: (user: UserProfile) => void
  setAccessToken: (token: string) => void
  logout: () => void
}

// CRITICAL: Access Token lưu trong Zustand memory (không localStorage/sessionStorage)
// Refresh Token lưu trong HTTP-only cookie (backend set tự động)
// Khi refresh page → gọi /auth/refresh để lấy lại Access Token
```

---

## Forms & Validation

```typescript
// Dùng React Hook Form + Zod cho tất cả forms
const schema = z.object({
  title: z.string().min(1, 'Tiêu đề không được bỏ trống').max(200),
  priority: z.enum(['LOW', 'MEDIUM', 'HIGH', 'URGENT']),
  deadline: z.string().datetime().optional(),
})

const form = useForm<z.infer<typeof schema>>({
  resolver: zodResolver(schema),
})
```

---

## Styling — Tailwind v4

```typescript
// Tailwind v4: dùng @import "tailwindcss" trong globals.css
// KHÔNG còn dùng tailwind.config.js — config trong CSS

// shadcn/ui: dùng cn() utility để merge classes
import { cn } from '@/lib/utils'

<div className={cn('base-class', condition && 'conditional-class', className)} />
```

---

## WebSocket (Socket.io client)

```typescript
// Kết nối Socket.io để nhận real-time notifications
// Tự động reconnect khi mất kết nối
// Join room theo userId sau khi auth:
socket.emit('join', { userId: user.id })

// Listen events:
socket.on('notification:new', (notification) => {
  notificationStore.addNotification(notification)
})
```

---

## Component Patterns

```typescript
// ✅ Props interface cho mọi component
interface TaskCardProps {
  task: Task
  onUpdate: (id: string, data: Partial<Task>) => void
  className?: string  // luôn expose className để flexible
}

// ✅ Tách loading/error state thành component riêng
if (isLoading) return <TaskCardSkeleton />
if (error) return <ErrorMessage message={error.message} />

// ✅ Dùng Suspense + loading.tsx cho page-level loading
```

---

## Environment Variables
```
NEXT_PUBLIC_API_URL=http://localhost:3000/api/v1
NEXT_PUBLIC_SOCKET_URL=http://localhost:3000
NEXT_PUBLIC_APP_URL=http://localhost:3001
```

---

## DO NOT
- Đừng dùng `'use client'` trên layout hoặc page nếu không cần thiết
- Đừng fetch data trong Client Component nếu có thể fetch ở Server Component
- Đừng lưu Access Token trong `localStorage` hay `sessionStorage` — dùng Zustand memory
- Đừng gọi API trực tiếp bằng `fetch` trong component — dùng service functions
- Đừng import shadcn/ui components và sửa file trong `components/ui/` — extend bằng component mới
- Đừng dùng `<img>` — dùng `<Image>` của Next.js
- Đừng dùng CSS modules — dùng Tailwind classes

## Slash Commands Available
- `/plan` — Lên kế hoạch trước khi implement một trang/feature
- `/tdd` — Implement với test (React Testing Library)
- `/code-review` — Review UI + accessibility + performance
- `/build-fix` — Sửa build/hydration errors có hệ thống
