# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Thái Phúc Tiến` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `202602873` |
| GitHub | `tom-e666` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `6771 / 7125` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.279 | 20.4 % | BLOCK |
| IDM | 1.536 | 0.1 % | (baseline) |
| EgoMLP | 0.776 ± 0.003 | 15.2 .. 28.5 % | BLOCK |
| PercMLP | 0.632 ± 0.002 | 0.3 .. 2.7 % | BLOCK |
| PercMLP+ (sau khi sửa) | 0.832 ± 0.019 | 0.2 .. 0.2 % | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Khi chạy Bước 0 thì em thấy bộ dữ liệu thô có 9993 quỹ đạo với 992679 dòng nhưng bị dính 383 quỹ đạo trùng nhau hoàn toàn nên phải lọc bỏ đi còn lại 9610 quỹ đạo sạch. Ngoài ra cột Acc_LV đo gia tốc xe trước rất vô dụng vì tính tương quan với đạo hàm vận tốc chỉ ra 0.045 (trong khi xe mình Acc_FAV tận 0.929), nghĩa là số đo cảm biến này bị nhiễu hoặc sai số nặng, nên phải tự tính lại gia tốc mượt a_lead bằng cách lấy đạo hàm từ Speed_LV. Việc chia tập train/test theo từng quỹ đạo thay vì chia bừa từng dòng là để tránh bị rò rỉ dữ liệu (data leakage), vì các bước thời gian liên tiếp trong cùng một xe chạy rất giống nhau, nếu chia theo dòng thì model sẽ học thuộc lòng luôn.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 làm khác bit ở toàn bộ 600 đoạn (100%), nhưng không có đoạn nào bị vượt dung sai cả vì sai số lớn nhất chỉ có 1.2e-04 m/s^2 (bé hơn nhiều so với atol = 0.001). Khi bật tính năng phanh khẩn cấp AEB thì có đúng 4 đoạn bị lệch vượt dung sai là đoạn 218, 331, 477 và 538 với độ lệch to nhất tới 1.425 m/s^2. Em mở các đoạn này ra xem thì thấy xe trước phanh gấp làm thời gian va chạm TTC rớt xuống dưới 1.5 s nên hệ thống AEB nhảy vào can thiệp đạp phanh tối đa. Còn vụ phát lại thì chế độ fastest chạy siêu nhanh nhưng lờ đi thời gian thực, báo 0 frame trễ và 0 frame rớt trên 59895 frame; trong khi bật realtime lên thì lộ ra ngay 584 frame bị trễ và 575 frame bị rớt khiến 379 đoạn bị đổi lệch lệnh lái.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản của em chạy với tốc độ xe là 14.0 m/s, khoảng cách từ 2.76 m đến 59.31 m, chia thành 60 x 45 tức là 2700 ô. Khi xe phản ứng tức thì trễ 0.0 s thì có 3.6% số ô là va chạm bất khả kháng do quá gần. Nhưng khi tăng độ trễ lên 0.6 s thì vùng không thể tránh này tăng vọt lên 10.2%, làm cho xe IDM cũng bị đâm tăng từ 3.6% lên đúng 10.2% (tăng thêm 6.6%). Dữ liệu log thực tế không hề bao phủ được các tình huống khó này, vì tài xế con người trong log phanh gắt nhất ở phân vị 99.9 chỉ tới 3.62 m/s^2 (trong khi phanh xe có thể đạt tới 9.0 m/s^2), và trên lưới có tận 12.7% số ô đòi hỏi phải phanh mạnh hơn mức con người từng phanh trong log.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng điểm L2 và va chạm của 4 bộ điều khiển:
| Planner | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín (%) | Hạng CL |
|---|---|---|---|---|
| const-vel | 1.279 | 3 | 20.4 % | 3 |
| IDM | 1.536 | 4 | 0.1 % | 1 |
| EgoMLP | 0.776 | 2 | 22.1 % | 4 |
| PercMLP | 0.632 | 1 | 1.4 % | 2 |

Các cặp bị đảo lộn thứ tự là: ('const-vel', 'IDM'), ('const-vel', 'EgoMLP'), ('IDM', 'EgoMLP'), ('IDM', 'PercMLP').
IDM có L2 tệ nhất vì nó là luật vật lý giữ khoảng cách an toàn, nó không thèm học theo thói quen bám sát đuôi của con người trong log nên sai số L2 to, nhưng vào vòng kín tự lái thì lại an toàn nhất với 0.1% va chạm. Còn EgoMLP có L2 tốt hơn const-vel (0.776 m so với 1.279 m) nhưng vào vòng kín lại đâm nhiều nhất tới 22.1% vì nó không nhìn thấy xe phía trước (mù gap), khi xe trước phanh thì nó không biết đường phanh lại, cộng thêm sai số tích lũy qua từng bước làm xe trôi dạt rồi đâm.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP chạy qua các seed dao động rất mạnh, từ 15.2% lên tới 28.5% qua 5 seed (lệch nhau tận 13.3%). Khoảng tin cậy Wilson 95% ở seed xấu nhất là [26.9%, 30.2%], còn seed tốt nhất là tầm [14.0%, 16.6%]. Nếu em chỉ chạy hên xui đúng 1 seed trúng vào cái 15.2% rồi mang đi nộp báo cáo thì sẽ tưởng mô hình lái khá ổn, nhưng thực ra sang seed khác nó đâm tới 28.5%. Điều này cực kỳ nguy hiểm trong thực tế xe tự hành vì sự may rủi khi khởi tạo trọng số ngẫu nhiên ban đầu sẽ che giấu lỗi của mô hình, ra đường thật gặp trường hợp xấu xe sẽ đâm liên tục mà mình không lường trước được.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tỷ lệ va chạm ở trễ 0.6 s:
| Bộ điều khiển | Xe sau phát lại (%) | Xe sau phản ứng (%) | Va chạm phía sau (Replay / React) |
|---|---|---|---|
| const-vel | 80.6 % | 80.6 % | 0.0 % / 0.0 % |
| IDM | 80.9 % | 10.2 % | 70.6 % / 0.0 % |
| EgoMLP | 82.5 % | 82.5 % | 0.0 % / 0.0 % |
| PercMLP | 78.9 % | 35.3 % | 43.9 % / 0.0 % |

Tham số T hiệu chỉnh được là T = 1.4 s với khoảng cách Wasserstein W1 = 0.34 (khá sát với mức 1.85 s của người thật, tốt hơn nhiều mức W1 = 0.89 khi xe không phản ứng). Khi xe sau chỉ phát lại thì nó cứ phi đều theo log, nên khi IDM phanh gấp tránh xe trước thì xe sau húc thẳng vào đuôi IDM tới 70.6% (đây là va chạm giả do mô phỏng dở chứ không phải IDM lái lỗi). Khi cho xe sau biết phản ứng thì va chạm sau biến mất còn đúng 0.0%, tổng va chạm của IDM tụt từ 80.9% xuống 10.2%, giúp trả lại công bằng cho IDM là thuật toán lái chuẩn nhất.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị cổng chặn lại (BLOCK) vì trượt tận 5 quy tắc: R2 (va chạm log 2.7%, cận trên 3.3% > IDM 0.1% + 1.0%), R3 (phanh gấp 10.8%, cận trên 15.8% > IDM 0.5% + 5.0%), R4 (lưới kịch bản đâm 25.1% > IDM 0.0% + 2.0%), R5 (seed dao động từ 0.3% đến 2.7% vượt mốc 1.0%), và R6 (xe sau phản ứng đâm 35.3% > IDM 10.2% + 2.0%). Nguyên nhân là do log thực tế thiếu các ca xe khác tạt đầu sát sạt và mô hình bị sai số tích lũy khi tự lái. Em sửa bằng cách nhồi thêm 196900 cửa sổ từ 20000 kịch bản cut-in mô phỏng và chạy 2 vòng DAgger (gần 900000 mẫu) để lấy IDM dạy lại cho mô hình cách tự xử lý khi đi lệch. Sau khi sửa thì PercMLP+ được duyệt RELEASE (va chạm log còn 0.2%, lưới đâm còn 0.6%). PercMLP+ lái mượt hơn IDM (L2 là 0.832 m so với 1.536 m) nhưng độ an toàn tuyệt đối vẫn thua IDM tí xíu (0.2% so với 0.1%). Chưa nên cho ra đường thật vội, em nghĩ cần đem lên xe chạy thử có tài xế ngồi kèm và test thêm các tình huống cảm biến bị nhiễu đã.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba hạn chế lớn nhất làm bài lab chưa đủ để đem ra chạy đường thật là:
1. Chỉ mô phỏng 1 chiều dọc (1D): Xe chỉ biết đi thẳng bám đuôi trên 1 làn, không biết đánh lái né vật cản, không tính tới đường trơn trượt mưa gió hay dốc cua gì cả.
2. Cảm biến hoàn hảo như hack: Mô hình được cấp luôn khoảng cách gap và tốc độ xe trước lead_v chính xác 100%, trong khi camera hay radar ngoài đời lúc nào cũng bị nhiễu, chậm vài mili-giây hoặc nhận diện nhầm bóng cây/vệt sáng.
3. Các xe xung quanh hành xử như robot: Xe khác trong bài đều chạy theo công thức IDM định sẵn, chưa thể hiện được sự bất cẩn, say xỉn hay tạt đầu ẩu của tài xế ngoài đời thực.
Để thu hẹp khoảng cách thực tế, bước tiếp theo cần dùng môi trường mô phỏng 3D xịn như CARLA, nạp thẳng dữ liệu ảnh camera thô vào model, đem xe ra chạy thử ở sa hình kín và cho chạy ngầm (shadow mode) trên xe thật để kiểm tra.
