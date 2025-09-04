# CFTemplatesRepo — Cafe Infra (CloudFormation + CI/CD)

用 **AWS CloudFormation** 定義並部署本專案的雲端基礎設施，包含：
- `cafe-network.yaml`：VPC、公有子網（供應用/EC2 使用）
- `cafe-app.yaml`：示範動態站台（EC2 + SG + UserData）
- `frontend-stack.yaml`：前端靜態架構（S3 + CloudFront），並把關鍵輸出寫入 **SSM Parameter Store**
- （此 repo 也可搭配 CodePipeline/CodeBuild 自動部署）

> 預設區域：**ap-northeast-1 (Tokyo)**  
> 建議在 **AWS CloudShell** 執行以下指令（內建 AWS CLI）。

---

## 目錄結構
templates/
├─ cafe-network.yaml # VPC / 公有子網（輸出 SubnetID / VpcID 並 Export）
├─ cafe-app.yaml # EC2 + SG（UserData 安裝 Apache/MariaDB/PHP）
└─ frontend-stack.yaml # S3 + CloudFront + SSM 參數（/cafe/frontend/*）

---

## 快速開始（CloudShell）

```bash
# 共用變數
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-1
REPO_ROOT=~/environment/CFTemplatesRepo

# 1) 部署網路層
aws cloudformation deploy \
  --stack-name cafe-network \
  --template-file $REPO_ROOT/templates/cafe-network.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# 2) 部署應用層（EC2）
aws cloudformation deploy \
  --stack-name cafe-app \
  --template-file $REPO_ROOT/templates/cafe-app.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# 3) 部署前端基礎設施（S3 + CloudFront）
SITE_BUCKET="cafe-frontend-${ACCOUNT_ID}-${REGION}"
aws cloudformation deploy \
  --stack-name the-cafe-frontend \
  --template-file $REPO_ROOT/templates/frontend-stack.yaml \
  --parameter-overrides SiteBucketName=${SITE_BUCKET} \
  --capabilities CAPABILITY_NAMED_IAM

部署完成後會有：
1.SSM 參數：
/cafe/frontend/bucket：網站 S3 桶名（例如 cafe-frontend-<acct>-ap-northeast-1）
/cafe/frontend/distribution：CloudFront Distribution ID
2.CloudFormation 輸出（在 the-cafe-frontend 堆疊的 Outputs 可看到 CloudFront 網域）


**前端 CI/CD（與 cathay-frontend repo 串接）**
流程
在 CodePipeline 建一個 pipeline（Source=GitHub，Build=CodeBuild）。
CodeBuild 專案 Source 選 CodePipeline，在前端 repo 放置 buildspec.yml（見該 repo）。
CodeBuild 服務角色需要下列最小權限（把 ACCOUNT_ID、REGION、SITE_BUCKET 改成你的）：
{
  "Version": "2012-10-17",
  "Statement": [
    // 讀 pipeline artifacts（由 CodePipeline 傳入）
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:GetObjectVersion","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::<YOUR-ARTIFACTS-BUCKET>",
        "arn:aws:s3:::<YOUR-ARTIFACTS-BUCKET>/*"
      ]
    },
    // 寫入前端網站桶
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket","s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::cafe-frontend-ACCOUNT_ID-REGION"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject","s3:DeleteObject","s3:PutObjectTagging","s3:DeleteObjectTagging"],
      "Resource": "arn:aws:s3:::cafe-frontend-ACCOUNT_ID-REGION/*"
    },
    // 建立 CloudFront 失效
    {
      "Effect": "Allow",
      "Action": ["cloudfront:CreateInvalidation","cloudfront:GetInvalidation","cloudfront:ListInvalidations"],
      "Resource": "arn:aws:cloudfront::ACCOUNT_ID:distribution/<DISTRIBUTION_ID>"
    },
    // 讀取 SSM 參數
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter","ssm:GetParameters","ssm:DescribeParameters"],
      "Resource": [
        "arn:aws:ssm:REGION:ACCOUNT_ID:parameter/cafe/frontend/bucket",
        "arn:aws:ssm:REGION:ACCOUNT_ID:parameter/cafe/frontend/distribution"
      ]
    }
  ]
}

flowchart LR
  user((End User)):::user

  %% === CDN Static Site Path ===
  user -- HTTPS --> cf[CloudFront Distribution]:::cf
  cf -- OAC (signed) --> s3[(S3 Site Bucket\ncafe-frontend-<account>-ap-northeast-1)]:::s3
  note over cf,s3: Bucket Policy 僅允許 CloudFront OAC 存取

  %% === Optional Dynamic/API Path ===
  user -- HTTP/HTTPS --> ec2[EC2 CafeInstance\nSecurityGroup: 80/22]:::ec2

  %% === Account / Region Context ===
  subgraph acct[AWS Account (ap-northeast-1)]
    ssm[(SSM Parameter Store\n/cafe/frontend/bucket\n/cafe/frontend/distribution)]:::ssm
    cf --- ssm
    subgraph vpc[ Cafe VPC ]
      igw[Internet Gateway]:::igw
      subgraph pub[Public Subnet]
        ec2
      end
      igw --- ec2
    end
  end

  classDef user fill:#fff,stroke:#555,color:#111
  classDef cf fill:#f6f8ff,stroke:#4c6ef5,stroke-width:1.5px
  classDef s3 fill:#fdf6ec,stroke:#f59f00
  classDef ec2 fill:#fff7f7,stroke:#fa5252
  classDef ssm fill:#eefbea,stroke:#37b24d
  classDef igw fill:#eee,stroke:#888
