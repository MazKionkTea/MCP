Tentu. Kalau tujuannya membuat MCP Server yang production-ready, mudah dikembangkan, dan bisa menangani banyak tool, saya sarankan arsitektur seperti ini.

1. Arsitektur yang saya rekomendasikan
                         ┌─────────────────────┐
                         │      User / App      │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │      LLM / AI       │
                         │ Claude / ChatGPT /  │
                         │ Cursor / Agent      │
                         └──────────┬──────────┘
                                    │
                              MCP Protocol
                                    │
                                    ▼
                  ┌─────────────────────────────────┐
                  │          MCP CLIENT              │
                  │                                 │
                  │  Tool discovery                 │
                  │  Tool invocation                │
                  │  Resource access                │
                  │  Prompt handling                │
                  └────────────────┬────────────────┘
                                   │
                                   ▼
              ┌─────────────────────────────────────────┐
              │              MCP SERVER                 │
              │                                         │
              │ ┌─────────────┐  ┌───────────────────┐ │
              │ │   Tools     │  │    Resources      │ │
              │ │             │  │                   │ │
              │ │ search()    │  │ documents         │ │
              │ │ create()    │  │ database data     │ │
              │ │ update()    │  │ configuration     │ │
              │ └──────┬──────┘  └─────────┬─────────┘ │
              │        │                   │           │
              │        └─────────┬─────────┘           │
              │                  ▼                     │
              │        ┌──────────────────┐            │
              │        │   Service Layer  │            │
              │        │                  │            │
              │        │ Business Logic   │            │
              │        │ Validation       │            │
              │        │ Authorization    │            │
              │        └────────┬─────────┘            │
              │                 │                      │
              │        ┌────────┴─────────┐            │
              │        ▼                  ▼            │
              │ ┌──────────────┐  ┌──────────────┐    │
              │ │ Repositories │  │ API Clients  │    │
              │ └──────┬───────┘  └──────┬───────┘    │
              └────────┼──────────────────┼────────────┘
                       │                  │
                       ▼                  ▼
                 ┌───────────┐      ┌─────────────┐
                 │ Database  │      │ External API │
                 │ PostgreSQL│      │ GitHub/etc.  │
                 └───────────┘      └─────────────┘


Prinsip penting: jangan memasukkan business logic langsung ke dalam MCP tool.

Contoh buruk:

tool → langsung query database → return


Lebih baik:

MCP Tool
   ↓
Service
   ↓
Repository / API Client
   ↓
Database / External API


Dengan begitu, MCP hanya menjadi interface untuk AI, bukan tempat seluruh aplikasi kamu hidup.

2. Workflow MCP

Misalnya user berkata:

"Cari customer bernama Budi dan tampilkan 5 transaksi terakhirnya."

Workflow-nya:

USER
 │
 │ "Cari customer Budi..."
 ▼
LLM
 │
 │ memahami intent
 ▼
MCP Client
 │
 │ list_tools()
 ▼
MCP Server
 │
 │
 │ tools:
 │ ├── search_customer
 │ └── get_transactions
 │
 ▼
LLM memilih:
search_customer
 │
 │ { "name": "Budi" }
 ▼
MCP Tool
 │
 ▼
CustomerService
 │
 ▼
CustomerRepository
 │
 ▼
PostgreSQL
 │
 │ customer_id = 123
 ▼
MCP Tool
 │
 ▼
LLM
 │
 │ sekarang tahu customer_id=123
 ▼
MCP Tool
get_transactions
 │
 ▼
TransactionService
 │
 ▼
PostgreSQL
 │
 ▼
LLM
 │
 ▼
USER
"Berikut 5 transaksi terakhir..."


Ada dua tahap reasoning:

Intent
  ↓
Tool selection
  ↓
Tool execution
  ↓
Result
  ↓
Next tool selection
  ↓
Tool execution
  ↓
Final answer


Ini sangat penting untuk agentic workflow.

3. Struktur folder yang saya rekomendasikan

Untuk project TypeScript/Node.js, saya akan menggunakan:

my-mcp-server/
│
├── src/
│   │
│   ├── index.ts
│   │
│   ├── server/
│   │   ├── mcp-server.ts
│   │   ├── transport.ts
│   │   └── context.ts
│   │
│   ├── tools/
│   │   ├── index.ts
│   │   │
│   │   ├── customer/
│   │   │   ├── search-customer.tool.ts
│   │   │   ├── get-customer.tool.ts
│   │   │   └── schemas.ts
│   │   │
│   │   ├── transaction/
│   │   │   ├── get-transactions.tool.ts
│   │   │   ├── create-transaction.tool.ts
│   │   │   └── schemas.ts
│   │   │
│   │   └── report/
│   │       ├── generate-report.tool.ts
│   │       └── schemas.ts
│   │
│   ├── resources/
│   │   ├── index.ts
│   │   ├── customer.resource.ts
│   │   └── report.resource.ts
│   │
│   ├── prompts/
│   │   ├── index.ts
│   │   └── customer-analysis.prompt.ts
│   │
│   ├── services/
│   │   ├── customer.service.ts
│   │   ├── transaction.service.ts
│   │   └── report.service.ts
│   │
│   ├── repositories/
│   │   ├── customer.repository.ts
│   │   └── transaction.repository.ts
│   │
│   ├── clients/
│   │   ├── github.client.ts
│   │   ├── stripe.client.ts
│   │   └── external-api.client.ts
│   │
│   ├── schemas/
│   │   ├── customer.schema.ts
│   │   └── transaction.schema.ts
│   │
│   ├── config/
│   │   ├── env.ts
│   │   └── config.ts
│   │
│   ├── auth/
│   │   ├── authorization.ts
│   │   └── permissions.ts
│   │
│   ├── utils/
│   │   ├── logger.ts
│   │   ├── errors.ts
│   │   └── pagination.ts
│   │
│   └── types/
│       └── index.ts
│
├── tests/
│   ├── tools/
│   ├── services/
│   ├── repositories/
│   └── integration/
│
├── .env
├── .env.example
├── .gitignore
├── package.json
├── tsconfig.json
├── README.md
└── Dockerfile

4. Fungsi setiap layer
server/

Tempat konfigurasi MCP.

server/
├── mcp-server.ts
├── transport.ts
└── context.ts


Tanggung jawab:

membuat MCP server
register tools
register resources
register prompts
mengatur transport
request context

Jangan taruh business logic di sini.

tools/

Ini adalah API yang dilihat oleh LLM.

Contoh:

tools/customer/search-customer.tool.ts


Secara konsep:

search_customer({
  name: string,
  limit?: number
})


Tool sebaiknya tipis:

Input
  ↓
Validation
  ↓
Service
  ↓
Output


Bukan:

Tool
 ├── validation
 ├── business logic
 ├── SQL
 ├── API call
 ├── formatting
 └── authorization

5. Service layer

Ini adalah otak aplikasi, bukan MCP.

Contoh:

services/
└── customer.service.ts


Misalnya:

class CustomerService {
  async searchCustomer(name: string) {
    // business rules

    return this.customerRepository.search(name);
  }
}


Keuntungan besar:

MCP Tool
    ↓
CustomerService


kemudian nanti bisa juga:

REST API
    ↓
CustomerService


atau:

Web App
    ↓
CustomerService


Jadi business logic tidak bergantung pada MCP.

6. Repository layer

Repository bertanggung jawab terhadap persistence.

CustomerService
       ↓
CustomerRepository
       ↓
PostgreSQL


Misalnya:

interface CustomerRepository {
  search(name: string): Promise<Customer[]>;
  findById(id: string): Promise<Customer | null>;
}


Dengan interface seperti ini kamu bisa mengganti:

PostgreSQL


menjadi:

MongoDB


tanpa mengubah MCP tool.

7. External API client

Kalau MCP kamu terhubung ke API eksternal, jangan panggil API langsung dari tool.

Buruk:

tool
 ↓
fetch("https://api.github.com/...")


Lebih baik:

tool
 ↓
service
 ↓
github.client
 ↓
GitHub API


Strukturnya:

clients/
├── github.client.ts
├── stripe.client.ts
└── slack.client.ts

8. Schema validation

Ini sangat penting karena input berasal dari model.

Misalnya tool:

create_customer


Input:

{
  "name": "Budi",
  "email": "budi@example.com"
}


Jangan percaya input LLM begitu saja.

Gunakan schema:

LLM
 ↓
MCP Tool
 ↓
Schema Validation
 ↓
Service


Misalnya dengan Zod:

const CreateCustomerSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email()
});


Dengan demikian input seperti:

{
  "name": "",
  "email": "hello"
}


langsung ditolak.

9. Authorization

Untuk MCP production, ini bagian yang sering dilupakan.

Jangan hanya berpikir:

"LLM boleh memanggil semua tool."


Lebih aman:

User
 ↓
Identity
 ↓
Authorization
 ↓
MCP Tool
 ↓
Service


Contohnya:

READ_CUSTOMER
CREATE_CUSTOMER
DELETE_CUSTOMER
EXPORT_DATA


Tool tertentu mungkin hanya boleh digunakan oleh role tertentu.

Contoh:

AI Agent
 ├── search_customer       ✓
 ├── get_customer          ✓
 ├── create_customer       ✓
 ├── delete_customer       ✗
 └── export_all_customers  ✗


Untuk operasi berbahaya, saya juga menyarankan human approval.

LLM
 ↓
delete_customer
 ↓
Authorization
 ↓
Human Approval
 ↓
Execute

10. Bedakan READ dan WRITE tools

Saya sangat menyarankan struktur seperti:

tools/
├── customer/
│   ├── search-customer.tool.ts   # READ
│   ├── get-customer.tool.ts      # READ
│   ├── create-customer.tool.ts   # WRITE
│   └── delete-customer.tool.ts   # DANGEROUS


Karena agent bisa melakukan autonomous tool calling.

Semakin besar dampak tool, semakin besar kontrol yang diperlukan.

Contoh:

READ
search_customer
     ↓
risiko rendah

WRITE
create_customer
     ↓
risiko sedang

DESTRUCTIVE
delete_customer
     ↓
risiko tinggi

11. Resource vs Tool

Gunakan Tool ketika AI perlu melakukan aksi.

search_customer()
create_invoice()
send_email()


Gunakan Resource ketika AI perlu membaca context/data.

customer://123
report://monthly/2026-08
file://README.md


Secara sederhana:

Tool     = "Do something"
Resource = "Give me something"
Prompt   = "Use this interaction template"

12. Arsitektur production

Kalau project sudah besar, saya akan naikkan menjadi seperti ini:

                         ┌─────────────┐
                         │    Users    │
                         └──────┬──────┘
                                │
                                ▼
                         ┌─────────────┐
                         │     LLM     │
                         └──────┬──────┘
                                │
                           MCP Client
                                │
                                ▼
                    ┌──────────────────────┐
                    │      MCP Server      │
                    │                      │
                    │  ┌───────────────┐   │
                    │  │ Tool Registry │   │
                    │  └───────┬───────┘   │
                    │          │            │
                    │  ┌───────▼────────┐   │
                    │  │ Validation      │   │
                    │  │ Authorization   │   │
                    │  │ Rate Limiting   │   │
                    │  └───────┬────────┘   │
                    │          │            │
                    │  ┌───────▼────────┐   │
                    │  │ Service Layer  │   │
                    │  └───────┬────────┘   │
                    └──────────┼────────────┘
                               │
               ┌───────────────┼───────────────┐
               │               │               │
               ▼               ▼               ▼
          PostgreSQL      External APIs      Redis
               │               │               │
               └───────────────┼───────────────┘
                               ▼
                       ┌──────────────┐
                       │ Observability│
                       │ Logs/Metrics │
                       │ Tracing      │
                       └──────────────┘

13. Workflow lengkap production

Saya akan menggunakan workflow:

                 ┌─────────────┐
                 │     User    │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │     LLM     │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │ MCP Client  │
                 └──────┬──────┘
                        ▼
              ┌────────────────────┐
              │ Authentication     │
              └─────────┬──────────┘
                        ▼
              ┌────────────────────┐
              │ Authorization      │
              └─────────┬──────────┘
                        ▼
              ┌────────────────────┐
              │ Input Validation   │
              └─────────┬──────────┘
                        ▼
              ┌────────────────────┐
              │      Tool          │
              └─────────┬──────────┘
                        ▼
              ┌────────────────────┐
              │      Service       │
              └─────────┬──────────┘
                        ▼
             ┌──────────────────────┐
             │ Repository / Client  │
             └──────────┬───────────┘
                        ▼
             ┌──────────────────────┐
             │ DB / External API    │
             └──────────┬───────────┘
                        ▼
              ┌────────────────────┐
              │ Output Validation  │
              └─────────┬──────────┘
                        ▼
                 ┌─────────────┐
                 │ MCP Client  │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │     LLM     │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │    User     │
                 └─────────────┘

14. Prinsip desain tool

Ada satu hal yang sangat penting:

Jangan membuat terlalu banyak tool yang sangat kecil.

Contoh kurang bagus:

get_customer_name()
get_customer_email()
get_customer_address()
get_customer_phone()


Lebih baik:

get_customer()


yang menghasilkan:

{
  "id": "123",
  "name": "Budi",
  "email": "budi@example.com",
  "phone": "...",
  "address": "..."
}


Namun jangan juga membuat tool "monster":

do_everything()


Idealnya satu tool merepresentasikan satu capability yang jelas.

15. Tool naming

Gunakan nama yang eksplisit.

Bagus:

search_customers
get_customer
create_customer
update_customer
delete_customer
get_customer_transactions


Hindari:

customer
data
execute
run
process
do_action


LLM lebih mudah memilih tool ketika nama dan deskripsinya jelas.

16. Output tool

Output juga sebaiknya terstruktur.

Jangan selalu mengembalikan:

"Customer Budi ditemukan dengan ID 123..."


Lebih baik:

{
  "customer": {
    "id": "123",
    "name": "Budi",
    "email": "budi@example.com"
  }
}


LLM bisa menginterpretasikan struktur tersebut dengan lebih konsisten.

17. Testing

Saya sarankan testing dibagi menjadi:

tests/
├── tools/
│   └── search-customer.test.ts
│
├── services/
│   └── customer.service.test.ts
│
├── repositories/
│   └── customer.repository.test.ts
│
└── integration/
    └── mcp-server.test.ts


Layer testing:

Unit Test
   ↓
Service Test
   ↓
Tool Test
   ↓
MCP Integration Test
   ↓
End-to-End Test

18. Observability

Untuk MCP production, log setiap tool invocation.

Contohnya:

{
  "request_id": "req_123",
  "tool": "search_customer",
  "user_id": "user_456",
  "duration_ms": 124,
  "status": "success"
}


Untuk tool yang sensitif, jangan log data rahasia.

Misalnya jangan:

password
API key
credit card
access token

19. Struktur final yang menurut saya paling ideal

Kalau saya sendiri yang memulai project MCP baru, saya akan mulai dengan struktur yang sedikit lebih sederhana:

mcp-server/
│
├── src/
│   ├── index.ts
│   │
│   ├── mcp/
│   │   ├── server.ts
│   │   ├── tools.ts
│   │   ├── resources.ts
│   │   └── prompts.ts
│   │
│   ├── tools/
│   │   ├── customer/
│   │   │   ├── search.ts
│   │   │   ├── get.ts
│   │   │   └── create.ts
│   │   └── transaction/
│   │       ├── list.ts
│   │       └── create.ts
│   │
│   ├── services/
│   │   ├── customer.ts
│   │   └── transaction.ts
│   │
│   ├── repositories/
│   │   ├── customer.ts
│   │   └── transaction.ts
│   │
│   ├── clients/
│   │   └── external-api.ts
│   │
│   ├── schemas/
│   │   ├── customer.ts
│   │   └── transaction.ts
│   │
│   ├── auth/
│   │   └── authorization.ts
│   │
│   ├── config/
│   │   └── env.ts
│   │
│   └── utils/
│       ├── logger.ts
│       └── errors.ts
│
├── tests/
├── .env.example
├── package.json
├── tsconfig.json
├── Dockerfile
└── README.md


Dan dependency flow-nya harus satu arah:

                 ┌─────────────┐
                 │ MCP / Tools │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │  Services   │
                 └──────┬──────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       ┌─────────────┐     ┌─────────────┐
       │ Repositories│     │ API Clients │
       └──────┬──────┘     └──────┬──────┘
              │                   │
              ▼                   ▼
          Database            External API


Jangan sampai:

Repository → Tool
Service → MCP
Database → Tool


karena nanti coupling-nya menjadi berantakan.

Rekomendasi saya

Kalau kamu baru mulai, jangan langsung membuat 20–50 folder. Mulai dengan:

MCP
 ↓
Tools
 ↓
Services
 ↓
Repositories / API Clients
 ↓
Infrastructure


Kemudian tambahkan auth, resources, prompts, observability, dan layer lain ketika kebutuhan muncul.

Kalau kamu ingin membangun MCP Server nyata, saya bisa lanjutkan dengan contoh TypeScript + Node.js + MCP SDK + PostgreSQL + Docker, termasuk kode lengkap, registrasi tools, schema Zod, authentication, error handling, dan struktur project yang siap dijalankan.
