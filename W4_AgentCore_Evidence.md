## Agent Core Bedrock

## L1
I. Tao S3 va upload 36 file md
1. AWS Console -> S3 -> Create bucket.
2. Đặt tên bucket -> Create bucket
3. Mở bucket vừa tạo -> Upload -> chọn tât cả 36 file trong knowledge_base/ -> Upload.
Evidence:
![S3-kb-upload-36 file md](./Evidence/s3-kb-36md.jpg)
II. Tao Bedrock Knowledge Base (KB)
1. AWS Console -> Amazon Bedrock
2. Ở thanh menu  -> Knowledge bases -> Create knowledge base.
3. chọn loại: Knowledge Base with vector store
4. Data source: S3 -> chon bucket vừa tạo ở bước I
5. IAM role: Create and use a new service role (wizard tạo role)
6. Embedding model: Amazon Titan Embeddings v2
![Embedding model](./Evidence/Embeđing%20module.jpg)
7. Vector store: Amazon OpenSearch Serverless (managed by Bedrock) -> Create
Evidence:
![KB tạo thành công](./Evidence/KB%20tao%20thanh%20cong.jpg)
III. Sync data source
1. Vào KB vừa tạo -> tab Data sources
2. Chọn data source rồi click Sync 
Evidence:
![Sync data source](./Evidence/Sync%20data%20source.jpg)
IV. Test retrieval (L1 readiness)
1. Vào KB rồi click Test Knowledge Base.
2. Chọn Retrieval and response generation: data sources and model
3. Chọn model Claude Sonnet 4.5
![Claude Sonnet 4.5](./Evidence/Claude%20Sonnet%204.5.jpg)
4. Đặt câu hỏi: "Who is the Team Platform lead?"
5. Kiểm Source chunks co team_platform.md.
6. Kiểm tra câu trả lời là: "Alex Chen"
7. Tiếp tục hỏi thêm 1 vài câu hỏi đại diện
Evidence:
![cau tra loi + source chunk 1](./Evidence/cau%20tra%20loi%20+%20source%20chunk%201.jpg)
----
![cau tra loi + source chunk 2](./Evidence/cau%20tra%20loi%20+%20source%20chunk%202.jpg)
----
![cau tra loi + source chunk 3](./Evidence/cau%20tra%20loi%20+%20source%20chunk%203.jpg)

## L2 Test conflict and multi-doc
1. Vào test Knowledge Base giống ở L1, kéo xuông phần Source -> Source chunks tăng từ 5 (để ở L1) tăng lên 9
![Tăng retrieval K](./Evidence/Tăng%20retrieval%20K.jpg)
2. Kéo xuống ở tab Configurations, phần Generation-> Generation prompt nhấn Edit rồi thêm nội dung như sau:
![Generation prompt](./Evidence/Generation%20prompt.jpg)
3. Ghi câu hỏi test conflict question: "What is PaymentGW's API rate limit?" — system phải retrieve cả v1 (500) lẫn v2 (1000) và xác định đúng v2 là current
![conflict question 1](./Evidence/conflict%20question%201.jpg)
----
![conflict question 2](./Evidence/conflict%20question%202.jpg)
4. Test với multi-doc question: "Can Team Commerce deploy on Friday night?" — system phải kết hợp deployment_policy.md + incident_response_policy.md + thông tin team
![multi-doc question 1](./Evidence/multi-doc%20question%201.jpg)
----
![multi-doc question 2](./Evidence/multi-doc%20question%202.jpg)
----
![multi-doc question 3](./Evidence/multi-doc%20question%203.jpg)

### L3 — Retrieval + Tools (Tool-Augmented RAG)
1. Seed Db:
1.1 Cài đặt Python về máy (nếu đã cái xong vào Power Shell sử dụng lệnh python --version -> ra được version VD: Python 3.14.4 là Ok)
1.2 mở terminal ở W4 cd vào phần scripts trong data_package
1.3 Tạo vene bằng lệnh: python -m venv .venv
                        .\.venv\Scripts\Activate.ps1
1.4 Cài deps từ pyproject.toml: pip install -U pip
                                pip install fastapi uvicorn psycopg2-binary pandas
1.5 Seed DB: python seed_data.py --db-type sqlite --sqlite-path geekbrain.db
uvicorn monitoring_api:app --port 8000
![Seed DB](./Evidence/Seed%20DB.jpg)
1.6 Có thể test 2 endpoint để xem API đã trả được dữ liệu chưa như sau: 
- http://127.0.0.1:8000/services
![endpoint services](./Evidence/endpoint%20services.jpg)
-  http://127.0.0.1:8000/metrics/PaymentGW
![endpoint PaymentGW](./Evidence/endpoint%20PaymentGW.jpg)

2. Tạo Lambda tools
2.1 Tạo Role cho Lambda
- Ở Console tìm và chọn IAM -> Role -> chọn Create role
- Trusted entity: AWS service
- Use case thì tìm và chọn Lambda -> Next
- Permissions: tìm vào tick AWSLambdaBasicExecutionRole -> Next
- Đặt tên Role name là gb-w4-lambda-role-an -> Create role
![Role cho Lambda](./Evidence/Role%20cho%20Lambda.jpg)
2.2 Thêm quyền tối thiểu
- Tạo 1 S3 bucket lưu riêng (geekbrain-db-anngo)
- Click vào role mới tạo, ở phần Permissions policies -> click Add permission -> Create inline policy
- Ở Policy editor chọn Json
![Policy tối thiểu](./Evidence/Policy%20tối%20thiểu.jpg)
2.3 Upload DB lên S3
- Vào S3 bucket geekbrain-db-anngo đã tạo trước đó vào upload file geekbrain.db
![geekbrain.db](./Evidence/geekbrain.db.jpg)
2.4 Tạo Lambda tool query_database
- Ở Console Tìm Lambda -> Create function
- Đặt tên Function (gb-query-database), chọn Runtime nên chọn cùng với phiên bản Python mình tải (3.14)
- click vào Additional settings, tick chọn Custom execution role -> Chọn role mà nãy đã tạo -> Save
- Create function
- Khi tạo xong click chọn tab Configuration -> Enviroment variables -> Edit và Add Environment variables với key và value như sau
![Add Environment variables](./Evidence/Add%20Environment%20variables.jpg)
- Quay lại phần code trong lambda và chỉnh sửa lại: 

import json
import os
import sqlite3
import boto3

s3 = boto3.client("s3")
DB_LOCAL_PATH = "/tmp/geekbrain.db"

def _download_db():
    bucket = os.environ["DB_BUCKET"]
    key = os.environ["DB_KEY"]
    s3.download_file(bucket, key, DB_LOCAL_PATH)

def _validate_sql(sql):
    sql_lower = sql.strip().lower()
    if not sql_lower.startswith("select"):
        raise ValueError("Only SELECT is allowed.")
    return sql

def lambda_handler(event, context):
    # Expect: {"sql": "..."}
    sql = event.get("sql")
    if not sql:
        return {"error": "Missing sql"}

    try:
        _validate_sql(sql)
        _download_db()

        conn = sqlite3.connect(DB_LOCAL_PATH)
        conn.row_factory = sqlite3.Row
        cur = conn.cursor()
        cur.execute(sql)
        rows = [dict(r) for r in cur.fetchall()]
        conn.close()

        return {"rows": rows}
    except Exception as exc:
        return {"error": str(exc)}

- sau đó nhấn Deploy
- Qua tab test thêm event: 
{"sql":"SELECT COUNT(*) as cnt FROM monthly_costs;"} -> Test 
![query_database thành công](./Evidence/query_database%20thành%20công.jpg)
->  Lambda gb-query-database chạy thành công, đã tải được geekbrain.db từ S3 và query được DB (COUNT = 36)

2.5 Tool 2: Service Metrics
- Cần phải URL public cho monitoring API -> sẽ sử dụng ngrok (nhanh không cần setup Ec2/lambda/API gateway , nhưng URL thay đổi khi restart; phụ thuộc máy local)
- Cài ngrok , mở ngrok sẽ có terminal mở ra
- gõ lệnh ngrok config add-authtoken YOUR_TOKEN (này kiếm ở acc mà đã tạo trên ngrok)
- gõ lệnh: ngrok http 8000 -> Ngrok sẽ chạy
![ngrok http 8000](./Evidence/ngrok%20http%208000.jpg)

- Vào Console -> lambda -> tạo 1 function tool thứ 2, với tên là gb-get-service-metrics và setup tương tự tool 1
- Vào tab Configuration -> Edit Enviroment variables:
![Edit Enviroment variables tool 2](./Evidence/Edit%20Enviroment%20variables%20tool%202.jpg)
- Quay lại phần code, thay vào đoạn code sau:

import json
import os
import urllib.request

def lambda_handler(event, context):
    service = event.get("service_name")
    if not service:
        return {"error": "Missing service_name"}

    base_url = os.environ["API_BASE_URL"].rstrip("/")
    url = f"{base_url}/metrics/{service}"

    try:
        with urllib.request.urlopen(url, timeout=10) as resp:
            data = json.loads(resp.read().decode("utf-8"))
        return {"metrics": data}
    except Exception as exc:
        return {"error": str(exc)}

-> Deploy, qua tab Test thay Event JSON vào: {"service_name":"PaymentGW"}
![Test Service Metrics](./Evidence/Test%20Service%20Metrics.jpg)

2.6 Tạo AgentCore Agent và Action Groups
1) Tạo Agent
- Ở Console -> Bedrock , tìm và chọn tab Agent ở thanh menu bên trái -> Create agent với tên là gb-agent-w4 -> created
- Ở phần Select model -> Chọn Claude Sonnet 4.5
- Gắn Knowledge Base đã tạo -> add (lưu ý hãy chọn Create and use a new service role từ trước và chọn model thì hãy nhấn Save rồi mới add được KB)
-> Save and exit
- Chọn test ở Agent vừa mới tạo -> prepared và test thử
![prepared và test thử](./Evidence/prepared%20và%20test%20thử.jpg)

2) Tạo Action Groups
- Chọn Edit in Agent Builder, ở phần Action groups -> Add
- Name: db_query , ở phần Action group invocation chọn Select an existing Lambda function và chọn Lambda Query (tool đã làm)
- Phần Action group function 1 -> chọn JSON Editor và sử dụng đoạn schema sau:
{
  "type": "object",
  "properties": {
    "sql": { "type": "string" }
  },
  "required": ["sql"]
}
-> Create
- Chọn tiếp Add Action group, nhập tên service_metrics
- Chọn lambda metrics đã tạo, Json Editor:
{
  "type": "object",
  "properties": {
    "service_name": { "type": "string" }
  },
  "required": ["service_name"]
}
-> Create
![Tạo Action Groups](./Evidence/Tạo%20Action%20Groups.jpg)
-> Save and Exit

3) Add permission trên Lambda
- Vào lần lượt 2 lambda đã tạo -> Configuration -> Permissions
- Kéo xuống phần Resource-based policy → Add permissions
![Add permissions Lambda](./Evidence/Add%20permissions%20Lambda.jpg)
----
![permissions Lambda1](./Evidence/permissions%20Lambda1.jpg)
----
![permissions Lambda2](./Evidence/permissions%20Lambda2.jpg)





