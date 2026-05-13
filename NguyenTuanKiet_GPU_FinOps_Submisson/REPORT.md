# Báo Cáo GPU FinOps Lab

**Sinh viên**: Nguyễn Tuấn Kiệt  
**MSSV**: 2A202600233  
**Ngày nộp**: 13/05/2026

---

## 1. Giới thiệu

### 1.1 Mục tiêu của bài lab
Bài lab này nhằm giúp sinh viên hiểu rõ về GPU FinOps - một lĩnh vực quan trọng trong quản lý chi phí GPU và tối ưu hóa tài nguyên điện toán. Thông qua các phần thực hành, sinh viên sẽ:
- Học cách giám sát và quản lý GPU cluster
- Hiểu về cost tracking và billing
- Áp dụng spot instances để tiết kiệm chi phí
- Cấu hình autoscaling tương tự KEDA
- Phân tích và tối ưu hóa chi phí
- Thực hành training model trên GPU thực tế
- Phân tích advanced optimization strategies

### 1.2 Tổng quan về GPU FinOps
GPU FinOps là sự kết hợp giữa Financial Operations (FinOps) và GPU computing. Nó tập trung vào:
- **Cost Visibility**: Hiểu rõ chi phí GPU đang sử dụng
- **Cost Optimization**: Tìm cách giảm chi phí mà không ảnh hưởng đến performance
- **Resource Efficiency**: Sử dụng tài nguyên GPU một cách hiệu quả nhất
- **Workload Management**: Quản lý các workload để tối ưu hóa chi phí

---

## 2. Phân tích từng phần

### Part 1: GPU Cluster Monitoring

#### 2.1.1 Kết quả quan sát
- **Cell 3 - View Cluster Nodes**: 
  - Số lượng GPU nodes: 7 nodes
  - Loại GPU: T4, A100, V100 (mixed)
  - Trạng thái cluster: Tất cả nodes hoạt động bình thường

- **Cell 4 - Cluster Metrics Summary**:
  - Tổng GPU: 14 GPUs
  - Busy GPUs: 11/14
  - Idle GPUs: 3
  - Memory tổng: 384.0 GB
  - Memory sử dụng: 256.4 GB
  - Utilization rate: 54.0%
  - Total Power Draw: 1239 W

#### 2.1.2 Insights
Cluster đang hoạt động ở mức utilization trung bình (54%). Có 3 GPU idle, cho thấy có cơ hội tối ưu hóa bằng cách consolidate workloads. Các node A100 có utilization cao (71-84%), trong khi T4 và V100 ở mức trung bình (65-72%). Power draw 1239W là hợp lý cho 14 GPUs. Cluster có khả năng xử lý thêm workloads mà không cần scale up ngay lập tức.

---

### Part 2: Workload Submission & Cost Tracking

#### 2.2.1 Kết quả thực hành
- **Cell 5 - Submit multiple workloads**:
  - Số lượng workloads submitted: 4 workloads
  - Trạng thái các workloads: 
    - train-resnet-001: running (node-06, GPU 0)
    - train-bert-002: running (node-02, GPU 1)
    - inference-api-003: running (node-06, GPU 1)
    - train-llm-004: queued
  - Thời gian execution: Immediate
  - Busy GPUs sau: 14/14 (100%)
  - Utilization: 69.5%

- **Cell 6 - Record billing for workloads**:
  - Tổng chi phí: $3.9229
  - Chi phí từng workload:
    - train-resnet-001 [ON-DEMAND]: $0.0292
    - train-bert-002 [ON-DEMAND]: $0.6117
    - inference-api-003 [SPOT]: $0.0035 (saved $0.0082)
    - train-llm-004 [SPOT]: $0.5505 (saved $1.2845)
  - Tổng tiết kiệm: $4.1229
  - Budget Used: 3.9%
  - Billing Status: OK

#### 2.2.2 Observations
Workload submission thành công với mix on-demand và spot instances. Spot instances đã tiết kiệm $4.1229 so với on-demand pricing, tương đương 51% tiết kiệm. Budget utilization chỉ 3.9% cho thấy project còn rất nhiều budget. Việc sử dụng spot cho inference-api-003 và train-llm-004 là quyết định tốt vì chúng có fault tolerance cao.

---

### Part 3: Spot Instance Management

#### 2.3.1 Kết quả phân tích
- **Cell 7 - Check spot pricing**:
  - Giá spot T4: $0.2615/hr (25.3% discount)
  - Giá spot A100: $2.7073/hr (26.2% discount)
  - Giá spot V100: $1.9399/hr (21.8% discount)
  - Availability: T4 (high), A100 (high), V100 (low)

- **Cell 8 - Request spot instances**:
  - Số lượng spot instances requested: 3
  - Success rate: 100% (3/3 granted)
  - Trạng thái: 
    - spot-t4-001 (T4): ✅ granted
    - spot-t4-002 (T4): ✅ granted
    - spot-a100-001 (A100): ✅ granted

- **Cell 9 - Simulate spot preemption**:
  - Số instances bị preempt: 1 (spot-t4-002)
  - Instances còn active: 5
  - Thời gian chạy trước preemption: 1s
  - Warning time: 120s
  - Spot cost: $0.0512
  - On-demand equivalent: $0.1707
  - Tổng savings: $0.1195 (70.0% tiết kiệm)

#### 2.3.2 Spot Instance Savings Analysis
Spot instances cung cấp tiết kiệm đáng kể (21-26% discount). Mặc dù có preemption risk, việc sử dụng spot cho fault-tolerant workloads vẫn rất hiệu quả. Trong test này, 1 instance bị preempt nhưng vẫn tiết kiệm 70% so với on-demand. Strategy tối ưu là sử dụng spot cho inference, batch processing, và training jobs có checkpoint, trong khi giữ on-demand cho critical services.

---

### Part 4: Autoscaling (KEDA-like)

#### 2.4.1 Kết quả cấu hình
- **Cell 10 - View and update autoscaling policy**:
  - Policy trước update:
    - scale_up_threshold: 70.0%
    - scale_down_threshold: 25.0%
    - cooldown_seconds: 30
    - max_nodes: 10
    - min_nodes: 2
    - preferred_gpu_type: T4
    - cost_aware: True
  - Policy sau update: ✅ Updated successfully

- **Cell 11 - Trigger autoscaler evaluation**:
  - Số cycles: 5 cycles
  - Current utilization: 69.5%
  - Actions thực hiện: 
    - Cycle 1: no_action (Util: 69.5%, Nodes: 7→7)
    - Cycle 2: no_action (Util: 69.5%, Nodes: 7→7)
    - Cycle 3: no_action (Util: 69.5%, Nodes: 7→7)
    - Cycle 4: no_action (Util: 69.5%, Nodes: 7→7)
    - Cycle 5: no_action (Util: 69.5%, Nodes: 7→7)
  - Kết quả scaling: Không cần scale (utilization trong range [25%-70%])

#### 2.4.2 Autoscaling Behavior Analysis
Autoscaling policy hoạt động đúng như thiết kế. Utilization 69.5% nằm trong safe range [25%-70%], nên không trigger scale-up. Cost-aware mode được bật giúp tối ưu hóa chi phí khi scaling. Cooldown 30s ngăn chặn thrashing. Policy này phù hợp cho workload hiện tại - nó sẽ scale up khi utilization vượt 70% và scale down khi dưới 25%.

---

### Part 5: Cost Analysis & Optimization

#### 2.5.1 Kết quả phân tích
- **Cell 12 - Take cost snapshots**:
  - Số snapshots: 5 snapshots
  - Waste percentage: 0.0% (tất cả snapshots)
  - Total cost/snapshot: $0.043889
  - Idle cost: $0.000000
  - Trend: Ổn định, không có biến động

- **Cell 13 - Waste Report**:
  - Average Waste: 3.9%
  - Tổng Idle Cost: $0.016609
  - Total Cost: $0.433055
  - Potential Monthly Save: $430.51
  - Severity: LOW

- **Cell 14 - Get Optimization Recommendations**:
  - Số recommendations: 2 recommendations
  - Top recommendations: 
    1. 🟡 [MEDIUM] USE_SPOT: Switch fault-tolerant workloads to spot instances for 60-70% savings (Estimated: 65.0%)
    2. 🟢 [LOW] SCHEDULING: Schedule non-urgent training jobs during off-peak hours for lower spot prices (Estimated: 20.0%)

- **Cell 15 - Full Dashboard View**:
  - Cluster: 14 GPUs across 7 nodes
  - Utilization: 69.5% | Busy: 14 | Idle: 0
  - Billing: $3.9229 / $100.00 budget
  - Alert Status: OK
  - Spot Savings: $0.1405 (70.0%)
  - Waste: 3.9% | Severity: LOW

#### 2.5.2 Waste Analysis và Recommendations
Waste analysis cho thấy cluster đang hoạt động rất hiệu quả với chỉ 3.9% waste. Không có idle GPUs, tất cả resources đang được sử dụng. Potential monthly savings $430.51 nếu áp dụng recommendations. Hai recommendation chính là:
1. Sử dụng spot instances cho fault-tolerant workloads (65% savings)
2. Scheduling jobs vào off-peak hours (20% savings)

Kết hợp cả hai strategy có thể tiết kiệm tới 85% chi phí cho các workloads phù hợp.

---

### Part 6: Visualization

#### 2.6.1 Biểu đồ được tạo
- **finops_cost_breakdown.png**: 
  - Mô tả: 3 subplots hiển thị cost breakdown theo GPU type, cost distribution, và cost trends
  - Insights: Cost phân bổ đều giữa các GPU types, không có concentration risk

- **finops_timeseries.png**:
  - Mô tả: Stackplot hiển thị cost theo thời gian (10 snapshots) và waste percentage
  - Trends: Cost ổn định, waste percentage luôn dưới 5%

#### 2.6.2 Visualization Analysis
Các biểu đồ cho thấy cost distribution rất balanced giữa các GPU types. Không có spike hoặc anomaly trong cost trends. Waste percentage luôn thấp (0-5%), chứng tỏ cluster được quản lý tốt. Time-series visualization giúp dễ dàng phát hiện các pattern hoặc anomaly nếu có.

---

### Part 7: Complete FinOps Workflow

#### 2.7.1 Kết quả workflow
- **Cell 18 - Full FinOps Optimization Workflow**:
  - 7 steps thực hiện:
    1. Initial cluster state: GPUs: 14 | Util: 69.5% | Idle: 0
    2. Submitting heavy workloads: After load: Util: 69.5% | Busy: 14/14
    3. Autoscaler evaluation: Decision: no_action - Utilization 69.5% within thresholds [25.0-70.0%]
    4. Cost analysis: Total cost/interval: $0.043889 | Waste: 0.0%
    5. Recommendations: [MEDIUM] USE_SPOT (~65% savings), [LOW] SCHEDULING (~20% savings)
    6. Spot instance optimization: Applied
    7. Final state: Optimized cluster with cost savings

#### 2.7.2 Workflow Analysis
Full FinOps workflow thực hiện thành công tất cả 7 bước. Workflow cho thấy quy trình hoàn chỉnh từ monitoring → workload submission → autoscaling → cost analysis → recommendations → optimization. Cluster duy trì utilization cao (69.5%) mà không cần scale up, chứng tỏ autoscaling policy hiệu quả. Recommendations được đưa ra dựa trên actual data, không phải heuristics.

---

### Part 8: Real GPU Workload Training

#### 2.8.1 GPU Detection & Setup
- **Cell 19 - Install dependencies & detect real GPU**:
  - GPU detected: Tesla T4
  - GPU memory: 15.6 GB
  - CUDA version: 12.8
  - Pricing: $0.35/hr (on-demand)
  - pynvml: available ✅
  - Status: Ready for training

#### 2.8.2 GPU Metrics Collection
- **Cell 20 - GPU Metrics Collection**:
  - Diagnostic test results: ✅ pynvml works
  - GPU utilization: 0% (idle)
  - Memory usage: 472/16106 MB (2.9%)
  - Power: 10.7W
  - Temperature: 38°C
  - Method: pynvml
  - Status: Ready for training

#### 2.8.3 FP32 Training Results
- **Cell 22 - Train FP32**:
  - Training time: 127.2s (3 epochs)
  - Accuracy progression: 34.8% → 54.1% → 64.9%
  - Peak GPU memory: 0.82 GB
  - Avg GPU utilization: 94.8%
  - Samples collected: 120
  - Cost: $0.012371

#### 2.8.4 AMP Training Results
- **Cell 23 - Train AMP**:
  - Training time: 64.5s (3 epochs)
  - Accuracy progression: 30.4% → 49.2% → 62.2%
  - Peak GPU memory: 0.60 GB
  - Avg GPU utilization: 88.6%
  - Avg power: 65.0W
  - Avg temperature: 77.6°C
  - Samples collected: 120
  - Cost: $0.006266

#### 2.8.5 FP32 vs AMP Comparison
- **Cell 24 - Compare FP32 vs AMP**:
  - Speed improvement: 1.97x faster (AMP)
  - Memory savings: 0.22 GB (26.8% reduction)
  - Cost savings: $0.006105 (49.4% reduction)
  - Accuracy difference: -2.7% (FP32 slightly better)
  - Power efficiency: AMP uses less power (65W vs higher for FP32)

#### 2.8.6 Real GPU Cost Report
- **Cell 25 - Report real GPU costs**:
  - FP32 workload reported: Cost $0.012400 | Rate $0.3500/hr
  - AMP workload reported (as spot): Cost $0.001900 | Saved $0.004400
  - Project: real-gpu-lab
  - Total Cost: $0.028700
  - Total Savings: $0.008700
  - Workloads: 4
  - Status: ✅ Successfully reported to gateway

#### 2.8.7 Generated Charts
- **real_gpu_comparison.png**: Comparison charts FP32 vs AMP (3 subplots)
- **real_gpu_telemetry.png**: GPU telemetry during training (4 subplots)
- **cost_per_epoch.png**: Cost breakdown per epoch

#### 2.8.8 Real GPU Training Analysis
FP32 vs AMP comparison cho thấy AMP là lựa chọn tối ưu cho training này:
- **Speed**: AMP nhanh gấp 1.97x, giảm training time từ 127.2s xuống 64.5s
- **Memory**: Tiết kiệm 0.22 GB (26.8%), quan trọng cho GPU có memory hạn chế
- **Cost**: Tiết kiệm 49.4% chi phí ($0.006105)
- **Accuracy**: Chỉ giảm 2.7%, có thể chấp nhận được cho hầu hết applications

Mặc dù AMP có accuracy thấp hơn, nhưng lợi ích về speed, memory, và cost rất đáng kể. Đây là best practice cho production training workloads.

---

### Part 8.5: Advanced GPU Cost Optimization

#### 2.9.1 Multi-GPU Cost Analysis
- **Cell 27 - Multi-GPU Cost Analysis**:
  - Scaling efficiency analysis:
    - 1 GPU: Speedup 1.00x, Time 2.00h, Cost $7.00, Efficiency 100.0%, Cost/Perf 7.00
    - 2 GPUs: Speedup 1.80x, Time 1.11h, Cost $7.78, Efficiency 90.0%, Cost/Perf 4.32
    - 4 GPUs: Speedup 3.20x, Time 0.62h, Cost $8.96, Efficiency 80.0%, Cost/Perf 2.80
  - Optimal configuration: 2-4 GPUs (best cost/performance ratio)

#### 2.9.2 Project Cost Forecasting
- **Cell 28 - Project Cost Forecasting**:
  - Phase 1 - Data Preparation:
    - GPU Type: T4
    - GPU Count: 1
    - Duration: 40 hours
    - Base Cost: $14.00
    - Best Case: $9.88 (with spot)
    - Worst Case: $18.12
    - Uncertainty: ±15.0%
  
  - Phase 2 - Model Training:
    - GPU Type: A100
    - GPU Count: 4
    - Duration: 120 hours
    - Base Cost: $1680.00
    - Best Case: $856.00 (with spot + AMP)
    - Worst Case: $2100.00
    - Uncertainty: ±15.0%

#### 2.9.3 Optimization Opportunity Analysis
- **Cell 29 - Optimization Opportunity Analysis**:
  - Baseline Training Cost: $1400.00
  - Prioritized Recommendations:
    1. Switch to Mixed Precision (AMP): $350.00 savings (25.0%) - LOW effort
    2. Use Spot Instances: $280.00 savings (20.0%) - MEDIUM effort
    3. Batch Size Optimization: $140.00 savings (10.0%) - LOW effort
  - Total potential savings: $770.00 (55.0%)

#### 2.9.4 Integrated Cost Dashboard
- **Cell 30 - Integrated Cost Dashboard**:
  - Dashboard layout: 2x3 (6 subplots)
  - Key metrics displayed:
    - Cost trends over time
    - Cost breakdown by GPU type
    - Waste analysis
    - Spot savings
    - Utilization patterns
    - Budget tracking
  - Status: ✅ Dashboard generated successfully
  - File: advanced_finops_dashboard.png

#### 2.9.5 Challenge Exercise
- **Cell 31 - Challenge Exercise**:
  - Scenario: Large Language Model Fine-tuning
  - Baseline: 8x A100 for 200h
  - Budget: $5000
  - Deadline: 2 weeks
  - Baseline Cost: $5840 (exceeds budget)
  - Optimization Strategy:
    1. Use Mixed Precision (AMP): -25% cost = $4380
    2. Use Spot Instances: -20% cost = $3504
    3. Optimize batch size: -10% cost = $3154
    4. Schedule during off-peak: -15% cost = $2681
  - Final Cost: $2681 (54% savings, within budget)
  - Implementation: Feasible within 2-week deadline

#### 2.9.6 Generated Advanced Charts
- **multi_gpu_scaling.png**: Multi-GPU scaling efficiency chart
- **project_forecast.png**: Project cost forecast with confidence intervals
- **optimization_roadmap.png**: Optimization roadmap and priority matrix
- **advanced_finops_dashboard.png**: Integrated dashboard with 6 key metrics

#### 2.9.7 Advanced Analysis
Advanced analysis cho thấy:
- **Multi-GPU Scaling**: Efficiency giảm khi tăng số GPUs (100% → 80%), nhưng cost/performance vẫn tốt hơn với 2-4 GPUs
- **Cost Forecasting**: Uncertainty ±15% là hợp lý, best case scenarios có thể đạt được với optimization
- **Optimization Opportunities**: Tổng tiết kiệm 55% có thể đạt được bằng cách kết hợp AMP, spot instances, và scheduling
- **Challenge Solution**: Có thể giảm cost từ $5840 xuống $2681 (54% savings) bằng cách áp dụng tất cả optimization strategies

Kết luận: Advanced optimization strategies rất hiệu quả và có thể giúp projects vượt qua budget constraints.

---

## 3. Kết luận và Học hỏi

### 3.1 Những kỹ năng FinOps đã học
Bài lab này cung cấp kinh nghiệm thực tế về các kỹ năng FinOps quan trọng:

1. **Cluster Monitoring**: 
   - Hiểu cách monitor GPU utilization, memory usage, power draw, và temperature
   - Biết cách phân tích cluster metrics để phát hiện bottlenecks
   - Kỹ năng: Real-time monitoring, metrics collection, anomaly detection

2. **Cost Tracking & Billing**:
   - Ghi nhận chi phí cho từng workload, phân biệt on-demand vs spot
   - Tính toán savings từ spot instances
   - Kỹ năng: Cost allocation, billing reconciliation, budget tracking

3. **Spot Instance Management**:
   - Hiểu spot pricing dynamics và discount rates (21-26%)
   - Quản lý preemption risk và implement fallback strategies
   - Kỹ năng: Spot bidding, preemption handling, risk management

4. **Autoscaling Configuration**:
   - Thiết lập autoscaling policies dựa trên utilization thresholds
   - Implement cost-aware scaling decisions
   - Kỹ năng: Policy design, threshold tuning, cost optimization

5. **Cost Analysis & Optimization**:
   - Phân tích waste (idle resources, underutilized GPUs)
   - Tạo recommendations dựa trên data
   - Kỹ năng: Waste analysis, root cause analysis, recommendation generation

6. **Real GPU Training Optimization**:
   - So sánh FP32 vs Mixed Precision (AMP)
   - Đo lường impact của optimization trên speed, memory, cost
   - Kỹ năng: Precision tuning, performance profiling, cost-benefit analysis

7. **Advanced Cost Forecasting**:
   - Dự báo chi phí cho multi-phase projects
   - Phân tích scaling efficiency
   - Kỹ năng: Forecasting, scenario analysis, optimization prioritization

### 3.2 Các chiến lược cost optimization hiệu quả

#### 3.2.1 Spot Instances
**Hiệu quả**: 21-26% discount, tổng tiết kiệm 70% khi kết hợp với on-demand
**Khi nên sử dụng**: 
- Batch processing jobs
- Training jobs có checkpoint
- Inference services với auto-recovery
- Non-critical workloads

**Best practices**:
- Luôn có fallback to on-demand
- Implement graceful shutdown (120s warning)
- Monitor preemption rates
- Combine với on-demand cho critical services

#### 3.2.2 Mixed Precision Training (AMP)
**Hiệu quả**: 1.97x faster, 26.8% memory savings, 49.4% cost reduction
**Khi nên sử dụng**:
- Deep learning training (CNN, Transformer, LLM)
- Inference workloads
- Bất kỳ workload nào có tolerance cho slight accuracy loss

**Best practices**:
- Kiểm tra accuracy impact (thường < 3%)
- Sử dụng gradient scaling để tránh underflow
- Monitor training stability
- Combine với batch size optimization

#### 3.2.3 Autoscaling
**Hiệu quả**: Tự động scale up/down dựa trên utilization, tiết kiệm idle resources
**Khi nên sử dụng**:
- Variable workloads
- Bursty traffic patterns
- Long-running services

**Best practices**:
- Set appropriate thresholds (25-70% range)
- Implement cooldown để tránh thrashing
- Enable cost-aware mode
- Monitor scaling decisions

#### 3.2.4 Workload Consolidation
**Hiệu quả**: Giảm idle GPUs, tăng utilization
**Khi nên sử dụng**:
- Khi có multiple small workloads
- Khi cluster có idle capacity

**Best practices**:
- Batch similar workloads
- Implement job scheduling
- Monitor resource contention

#### 3.2.5 Multi-GPU Optimization
**Hiệu quả**: Optimal configuration 2-4 GPUs (cost/performance ratio tốt nhất)
**Khi nên sử dụng**:
- Large training jobs
- Distributed inference

**Best practices**:
- Analyze scaling efficiency trước khi scale
- Không luôn scale to max GPUs
- Monitor communication overhead
- Balance cost vs speed

### 3.3 Ứng dụng thực tế trong projects

#### 3.3.1 Áp dụng trong production environments
Các kỹ năng học được có thể áp dụng trực tiếp:

1. **Monitoring & Alerting**:
   - Setup real-time GPU monitoring dashboard
   - Alert khi utilization < 25% hoặc > 80%
   - Track cost trends hàng ngày

2. **Cost Governance**:
   - Implement budget limits per project/team
   - Require cost estimates trước khi submit workloads
   - Monthly cost reviews

3. **Optimization Policies**:
   - Mandate AMP cho training jobs (25% savings)
   - Use spot instances cho batch jobs (20-30% savings)
   - Schedule non-urgent jobs off-peak (15% savings)

#### 3.3.2 Cost monitoring strategy
**Recommended approach**:

1. **Daily Monitoring**:
   - Track total cost, cost per GPU, cost per workload
   - Monitor waste percentage
   - Alert on anomalies

2. **Weekly Analysis**:
   - Review cost trends
   - Analyze top cost drivers
   - Identify optimization opportunities

3. **Monthly Planning**:
   - Forecast next month's cost
   - Plan optimization initiatives
   - Review budget vs actual

4. **Quarterly Review**:
   - Analyze year-to-date trends
   - Benchmark against industry standards
   - Plan long-term optimization roadmap

#### 3.3.3 Optimization roadmap
**Phase 1 (Immediate - Week 1-2)**:
- Enable AMP for all training jobs: 25% savings
- Implement spot instances for batch jobs: 20% savings
- Setup cost monitoring dashboard: visibility

**Phase 2 (Short-term - Month 1-2)**:
- Implement autoscaling: 10-15% savings
- Optimize batch sizes: 5-10% savings
- Schedule jobs off-peak: 15% savings

**Phase 3 (Medium-term - Month 3-6)**:
- Multi-GPU optimization: 5-10% savings
- Implement cost chargeback: accountability
- Advanced forecasting: planning

**Phase 4 (Long-term - Month 6+)**:
- Custom hardware selection: 10-20% savings
- Reserved instances: 30-40% savings
- Advanced ML-based optimization: 5-15% savings

**Expected total savings**: 60-80% over 6 months

---

## 4. Kết quả tổng hợp

### 4.1 Tóm tắt chi phí
- **Tổng chi phí mock cluster**: $3.9229
- **Tổng chi phí real GPU training**: $0.028700
  - FP32 training: $0.012400
  - AMP training: $0.001900
  - Spot savings: $0.004400
- **Tổng tiết kiệm từ optimization**: $4.1229 (mock) + $0.004400 (real) = $4.1269
- **Tổng tiết kiệm %**: 
  - Mock cluster: 51% (spot instances)
  - Real GPU: 49.4% (AMP + spot)
  - Combined: 50.7%

### 4.2 Key Takeaways
Những điểm chính học được từ bài lab:

1. **Spot instances là game-changer**: Tiết kiệm 21-26% giá, và khi kết hợp với on-demand có thể tiết kiệm 70%. Rủi ro preemption có thể quản lý được với proper fallback strategies.

2. **Mixed Precision (AMP) là must-have**: 1.97x faster, 26.8% memory savings, 49.4% cost reduction. Accuracy loss chỉ 2.7% là chấp nhận được cho hầu hết applications.

3. **Autoscaling cần cost-aware mode**: Không phải chỉ scale dựa trên utilization, mà cần xem xét chi phí. Cost-aware autoscaling giúp tránh over-provisioning.

4. **Waste analysis là critical**: Chỉ 3.9% waste trong lab này, nhưng trong production có thể cao hơn. Regular waste analysis giúp phát hiện optimization opportunities.

5. **Multi-GPU scaling không phải linear**: Efficiency giảm khi tăng số GPUs (100% → 80%), nên cần analyze trước khi scale. Optimal configuration thường là 2-4 GPUs.

6. **Forecasting giúp planning**: Dự báo chi phí với confidence intervals giúp plan budget và identify risks. Uncertainty ±15% là hợp lý.

7. **Combination strategies tạo impact lớn**: Kết hợp AMP (25%) + spot (20%) + scheduling (15%) = 60% tiết kiệm. Không có silver bullet, cần multi-pronged approach.

### 4.3 Khuyến nghị cho bài lab tiếp theo
Để nâng cao bài lab, có thể thêm:

1. **Real multi-GPU training**: Hiện tại chỉ test single GPU, nên thêm distributed training test
2. **Longer training duration**: Test autoscaling behavior với longer workloads
3. **Cost anomaly detection**: Implement ML-based anomaly detection
4. **Reserved instances**: Thêm reserved instances pricing model
5. **Custom hardware selection**: Cho phép chọn GPU type dựa trên cost-performance
6. **Real preemption simulation**: Simulate preemption giữa training để test recovery
7. **Team/project chargeback**: Implement cost allocation per team/project
8. **RI commitment planning**: Recommend khi nào nên buy reserved instances

---

## 5. Phụ lục

### 5.1 Danh sách file screenshots
Tất cả screenshots được lưu trong thư mục `screenshots/`:

**Part 1: GPU Cluster Monitoring**
- part1_cluster_monitoring.png - Cell 3 output (Cluster Nodes)
- part1_cluster_metrics.png - Cell 4 output (Cluster Metrics Summary)

**Part 2: Workload Submission & Cost Tracking**
- part2_workload_submission.png - Cell 5 output (Submit workloads)
- part2_billing_summary.png - Cell 6 output (Billing Summary)

**Part 3: Spot Instance Management**
- part3_spot_pricing.png - Cell 7 output (Spot Pricing)
- part3_spot_request.png - Cell 8 output (Request Spot Instances)
- part3_spot_preemption.png - Cell 9 output (Spot Preemption)

**Part 4: Autoscaling (KEDA-like)**
- part4_autoscaler_policy.png - Cell 10 output (Autoscaling Policy)
- part4_autoscaler_evaluation.png - Cell 11 output (Autoscaler Evaluation)

**Part 5: Cost Analysis & Optimization**
- part5_cost_snapshots.png - Cell 12 output (Cost Snapshots)
- part5_waste_report.png - Cell 13 output (Waste Report)
- part5_recommendations.png - Cell 14 output (Optimization Recommendations)
- part5_dashboard.png - Cell 15 output (Full Dashboard)

**Part 6: Visualization**
- part6_cost_breakdown_viz.png - Cell 16 output (Cost Breakdown)
- part6_timeseries_viz.png - Cell 17 output (Time-series Tracking)

**Part 7: Complete FinOps Workflow**
- part7_full_workflow.png - Cell 18 output (Full Workflow)

**Part 8: Real GPU Workload**
- part8_gpu_detection.png - Cell 19 output (GPU Detection)
- part8_gpu_metrics_diagnostic.png - Cell 20 output (GPU Metrics)
- part8_fp32_summary.png - Cell 22 output (FP32 Training)
- part8_amp_summary.png - Cell 23 output (AMP Training)
- part8_fp32_vs_amp_comparison.png - Cell 24 output (FP32 vs AMP)
- part8_real_gpu_cost_report.png - Cell 25 output (Cost Report)

**Part 8.5: Advanced GPU Cost Optimization**
- part85_multi_gpu_analysis.png - Cell 27 output (Multi-GPU Analysis)
- part85_project_forecast.png - Cell 28 output (Project Forecast)
- part85_optimization_analysis.png - Cell 29 output (Optimization Analysis)
- part85_integrated_dashboard.png - Cell 30 output (Integrated Dashboard)
- part85_challenge_strategy.png - Cell 31 output (Challenge Strategy)

### 5.2 Danh sách file charts
Tất cả generated charts được lưu trong thư mục `generated_charts/`:

- **finops_cost_breakdown.png** - Cost breakdown visualization (3 subplots)
- **finops_timeseries.png** - Time-series cost tracking (stackplot + waste %)
- **real_gpu_comparison.png** - FP32 vs AMP comparison charts
- **real_gpu_telemetry.png** - GPU telemetry during training (4 subplots)
- **cost_per_epoch.png** - Cost breakdown per epoch
- **multi_gpu_scaling.png** - Multi-GPU scaling efficiency chart
- **project_forecast.png** - Project cost forecast with confidence intervals
- **optimization_roadmap.png** - Optimization roadmap and priority matrix
- **advanced_finops_dashboard.png** - Integrated dashboard (6 subplots)

### 5.3 Notebook
- **gpu_finops_lab.ipynb** - Notebook đã chạy hoàn chỉnh với tất cả outputs

### 5.4 Tham khảo
**Tài liệu tham khảo chính**:
- OpenCost Documentation: https://www.opencost.io/
- KEDA Autoscaling: https://keda.sh/
- PyTorch Mixed Precision: https://pytorch.org/docs/stable/amp.html
- AWS Spot Instances: https://aws.amazon.com/ec2/spot/
- FinOps Foundation: https://www.finops.org/

**Công cụ sử dụng**:
- Pandas: Data analysis
- Matplotlib: Visualization
- Plotly: Interactive charts
- PyTorch: Deep learning framework
- pynvml: GPU monitoring

---

**Ngày hoàn thành**: 13/05/2026  
**Ghi chú**: 
- Tất cả 8.5 parts đã hoàn thành thành công
- Notebook chạy không lỗi, tất cả outputs được lưu
- Screenshots được chụp với header thông tin sinh viên hiển thị
- Tất cả generated charts được lưu đúng format PNG
- Real GPU training được thực hiện trên Tesla T4 (Kaggle/Colab)
- Tổng tiết kiệm chi phí: 50.7% (mock cluster + real GPU)
