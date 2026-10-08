# Báo cáo lab: chọn tracker cho 5 video

**Nhóm:** Nhóm 01 **Thành viên:** Đỗ Hoàng Quân - 2A202603016

Detector cố định: `yolo26n.pt`, ảnh 640 px, Re-ID `osnet_x0_25_msmt17`. Không đổi các mục này trong bài nộp chính.

## 1. Cấu hình đã chọn

| Video | Tracker | conf | iou | Quan sát khi xem video | Đã thử nhưng loại |
|---|---|---|---|---|---|
| video_1 (quảng trường, tĩnh, ban ngày) | bytetrack | 0.3 | 0.5 | Camera tĩnh, người đi thẳng giữ ID rất mượt. Lúc đi khuất sau gốc cây, tracker phục hồi ID tốt khi người vừa bước ra. Ít khi bị nhảy ID. | `strongsort` (conf 0.3, iou 0.5): Chạy rất chậm trên CPU do trích xuất Re-ID; kết quả không chênh lệch nhiều so với ByteTrack vì cảnh ít che khuất phức tạp. |
| video_2 (phố đêm, tĩnh, rất đông) | bytetrack | 0.25 | 0.5 | Mật độ người rất đông và nhỏ, hạ conf=0.25 giúp bắt được các bóng người ở xa/tối. ByteTrack phân bổ 2 giai đoạn giúp duy trì track liên tục của đám đông. | `botsort` (conf 0.3, iou 0.5): Bị đứt track nhiều người do đêm tối khiến Re-ID bị nhiễu màu sắc, đồng thời conf=0.3 làm mất người nhỏ ở xa. |
| video_3 (camera di động, ảnh nhỏ) | ocsort | 0.3 | 0.45 | Camera di chuyển và FPS thấp khiến vị trí người nhảy quãng lớn. OC-SORT dựa vào quan sát thực tế (observation-centric) để nội suy quỹ đạo nên ít bị mất track khi người đổi hướng. | `bytetrack` (conf 0.3, iou 0.5): Kalman Filter tuyến tính đơn giản của ByteTrack dự đoán sai vị trí ở frame tiếp theo do FPS thấp, dẫn đến ID switch liên tục. |
| video_4 (trong nhà, camera di chuyển) | botsort | 0.35 | 0.5 | Nâng conf lên 0.35 lọc sạch các bóng người ảo phản chiếu trên sàn gạch bóng và vách kính. Re-ID của BoT-SORT giúp phân biệt chính xác người thật với ảnh phản chiếu. | `bytetrack` (conf 0.25, iou 0.5): Conf thấp làm detector bắt nhầm hình phản chiếu trên kính thành người mới, sinh ra nhiều ID rác nhảy nhót. |
| video_5 (trên xe bus, giao lộ đông) | botsort | 0.3 | 0.5 | Góc nhìn rung lắc mạnh khi xe bus di chuyển qua ngã tư. Cơ chế bù chuyển động camera (CMC) của BoT-SORT giúp triệt tiêu rung giật, giữ ID người qua đường ổn định. | `ocsort` (conf 0.3, iou 0.5): Rung lắc quá mạnh của xe bus làm sai lệch momentum định hướng, dễ bị nhầm lẫn giữa người đi bộ và người đi xe máy. |

## 2. Số liệu video_1

Dán bảng HOTA / MOTA / IDF1 do `scripts/evaluate_practice.py` in ra.

```
HOTA: nhom_video1-pedestrian       HOTA      DetA      AssA      DetRe     DetPr     AssRe     AssPr     LocA      OWTA      HOTA(0)   LocA(0)   HOTALocA(0)
video_1                            26.912    15.068    48.13     15.288    82.599    50.467    84.841    84.551    27.116    32.559    81.385    26.498    
COMBINED                           26.912    15.068    48.13     15.288    82.599    50.467    84.841    84.551    27.116    32.559    81.385    26.498    

CLEAR: nhom_video1-pedestrian      MOTA      MOTP      MODA      CLR_Re    CLR_Pr    MTR       PTR       MLR       sMOTA     CLR_TP    CLR_FN    CLR_FP    IDSW      MT        PT        ML        Frag      
video_1                            17.292    82.526    17.356    17.932    96.889    11.29     14.516    74.194    14.158    3332      15249     107       12        7         9         46        44        
COMBINED                           17.292    82.526    17.356    17.932    96.889    11.29     14.516    74.194    14.158    3332      15249     107       12        7         9         46        44        

Identity: nhom_video1-pedestrian   IDF1      IDR       IDP       IDTP      IDFN      IDFP      
video_1                            25.713    15.236    82.32     2831      15750     608       
COMBINED                           25.713    15.236    82.32     2831      15750     608       

Count: nhom_video1-pedestrian      Dets      GT_Dets   IDs       GT_IDs    
video_1                            3439      18581     34        62        
COMBINED                           3439      18581     34        62        
```

`video_2` đến `video_5` không có nhãn trong gói lab. Không điền số cho các video đó.

## 3. Phân tích

### 1. Phân tích video_2 (Phố đêm, camera tĩnh trên cao, rất đông):
- **Quan sát & Hiệu quả:** Khi đối chiếu giữa ByteTrack và tracker có Re-ID (StrongSORT/BoT-SORT), ByteTrack (`conf=0.25`, `iou=0.5`) thể hiện sự vượt trội về độ ổn định danh tính trong cảnh này. Nhờ hạ ngưỡng tin cậy xuống 0.25 kết hợp với cơ chế liên kết 2 giai đoạn (liên kết cả detection điểm thấp), tracker không bị bỏ sót những người đi bộ có kích thước rất nhỏ và mờ nhạt trong ánh đèn đêm.
- **Tác động của ngữ cảnh:** Mặc dù mật độ giao cắt cao thường là lợi thế của Re-ID, nhưng cảnh quay ban đêm từ trên cao khiến ngoại hình người bị mất chi tiết màu sắc và chìm vào bóng tối. Điều này làm cho vector đặc trưng ngoại hình (Re-ID embedding từ OSNet) chứa nhiều nhiễu, dẫn đến việc tracker có Re-ID hay gán nhầm danh tính giữa các đối tượng tối màu. Trong trường hợp này, dựa vào mô hình chuyển động thuần túy và tận dụng detection điểm thấp của ByteTrack lại hiệu quả và ổn định hơn rất nhiều.

### 2. Phân tích video_3 (Camera di chuyển, độ phân giải thấp, FPS thấp):
- **Quan sát & Hiệu quả:** OC-SORT giữ ID tốt hơn rõ rệt so với ByteTrack tiêu chuẩn. Khi camera di chuyển và khung hình bị giật (low FPS), các đối tượng thường có bước nhảy tọa độ lớn giữa hai frame kề nhau, nhưng OC-SORT vẫn khóa đúng ID khi người tiếp tục di chuyển mà không bị gán ID mới.
- **Tác động của ngữ cảnh:** ByteTrack và SORT truyền thống sử dụng Kalman Filter với giả định vận tốc thay đổi mượt mà và tuyến tính. Khi FPS thấp và camera rung lắc, sai số tích lũy của Kalman Filter tăng vọt, dẫn đến việc dự đoán vị trí hộp ở frame tiếp theo bị trượt hoàn toàn so với bounding box thực tế (IoU = 0). OC-SORT khắc phục được điều này nhờ cơ chế *Observation-Centric* (tính toán lại vận tốc và quỹ đạo dựa trên các hộp quan sát thực tế khi track bị mất tạm thời), giúp thuật toán thích ứng vượt trội với các cảnh quay FPS thấp và camera di động.

## 4. Nếu có thêm thời gian

Nếu có thêm thời gian, nhóm sẽ:
1. Thử nghiệm thay thế mô hình trích xuất đặc trưng Re-ID nhẹ hơn hoặc fine-tune trên dữ liệu góc nhìn cao/ban đêm để cải thiện tính chính xác của BoT-SORT ở `video_2`.
2. Tinh chỉnh riêng ngưỡng `track_thresh` và `match_thresh` bên trong tracker (thay vì chỉ chỉnh `conf` của YOLO) để kiểm soát chặt chẽ hơn thời gian chờ phục hồi ID bị mất (max lost frames).
