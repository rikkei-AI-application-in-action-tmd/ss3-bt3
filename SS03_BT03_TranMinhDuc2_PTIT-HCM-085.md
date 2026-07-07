# Bài 3: Đọc hiểu & Dò lỗi qua Prompt (Code Tracing & AI Verification)

## 1. Đoạn phân tích lý do thất bại của prompt thô

Prompt thô "Code này có chạy được không?" đã thất bại trong việc bắt AI tìm lỗi logic biên vì:
1. **Thiếu vai trò (Role) và ngữ cảnh (Context):** Không cung cấp vai trò chuyên gia kiểm thử nên AI sẽ chỉ đóng vai trò là một công cụ kiểm tra cú pháp (syntax checker) thông thường.
2. **Câu hỏi dạng "Có/Không" đóng:** Câu hỏi quá chung chung dễ khiến AI đưa ra câu trả lời hời hợt. Đối với ngôn ngữ có tính chặt chẽ như Java, code biên dịch được và chạy bình thường ở trường hợp lý tưởng không có nghĩa là code đúng logic ở mọi trường hợp.
3. **Thiếu mục tiêu hướng tới (Goal) và ràng buộc biên (Constraint):** Lập trình viên không yêu cầu AI rà soát các trường hợp xấu, trường hợp biên (edge cases) như mảng rỗng hay null, do đó AI tự mặc định danh sách luôn có dữ liệu hợp lệ và bỏ qua nguy cơ ném ra `NullPointerException` hoặc `NaN` (Not a Number) khi chia cho 0.

## 2. Nội dung Prompt tối ưu mới

```text
Hãy đóng vai trò là một Chuyên gia Kiểm thử (QA/Tester) sắc bén và dày dạn kinh nghiệm về Java.

Mục tiêu là kiểm tra logic và tìm ra mọi lỗ hổng có thể làm sập chương trình trong đoạn mã Java tính trung bình sau:

```java
public class AverageCalculator {
    public static double calculateAverage(List<Integer> numbers) {
        int sum = 0;
        for (int num : numbers) {
            sum += num;
        }
        return (double) sum / numbers.size();
    }
}
```

Bối cảnh: Lập trình viên tập sự viết hàm trên nhưng chưa lường trước các trường hợp dữ liệu đầu vào không hợp lệ (edge cases).

Ràng buộc yêu cầu:
1. Phân tích và chỉ ra cụ thể các trường hợp biên nào (vd: null, rỗng) sẽ làm đoạn code trên phát sinh Exception gì (NullPointerException...).
2. Viết lại đoạn code Java hoàn chỉnh để khắc phục toàn bộ các lỗi biên đó (sử dụng kiểm tra null/empty và ném IllegalArgumentException).
3. Viết kèm một class JUnit 5 test case (AverageCalculatorTest) để kiểm thử cho cả trường hợp hợp lệ và các trường hợp biên vừa chỉ ra.

Định dạng: Trình bày phần phân tích lỗi bằng văn bản rõ ràng. Mã nguồn sửa đổi và mã JUnit test case phải đặt trong các khối code (code block) Java riêng biệt.
```

## 3. Đoạn code Java đã sửa lỗi và mã nguồn JUnit test case do AI sinh ra

```markdown
Dưới đây là phần phân tích lỗi và mã nguồn đã được khắc phục.

### Phân tích lỗi biên:
1. **`NullPointerException`:** Xảy ra nếu tham số `numbers` truyền vào là `null`. Vòng lặp `for-each` sẽ gọi phương thức `iterator()` trên một đối tượng `null`, làm ứng dụng văng lỗi. Vấn đề tương tự xảy ra do quá trình unboxing (tự động chuyển từ đối tượng Integer sang kiểu nguyên thủy int) trong vòng lặp nếu bên trong `numbers` có chứa phần tử mang giá trị `null`.
2. **Chia cho 0 (kết quả trả về `NaN`):** Xảy ra nếu `numbers` là một danh sách rỗng (`numbers.isEmpty()`). Biến `size()` sẽ bằng 0, phép chia `(double) sum / 0` trong Java sẽ không ném Exception mà sẽ trả về kết quả `NaN` (Not a Number), điều này gây sai lệch logic nghiệp vụ ở các tầng xử lý sau.

### Mã nguồn Java đã sửa lỗi:

```java
import java.util.List;

public class AverageCalculator {

    public static double calculateAverage(List<Integer> numbers) {
        // Kiểm tra trường hợp biên: danh sách bị null hoặc rỗng
        if (numbers == null || numbers.isEmpty()) {
            throw new IllegalArgumentException("Danh sách không được null hoặc rỗng");
        }

        int sum = 0;
        int validElementsCount = 0; // Đếm số phần tử không bị null

        for (Integer num : numbers) {
            // Bỏ qua phần tử null để tránh NullPointerException khi unboxing
            if (num != null) {
                sum += num;
                validElementsCount++;
            }
        }

        // Đề phòng trường hợp danh sách chỉ toàn giá trị null
        if (validElementsCount == 0) {
            throw new IllegalArgumentException("Danh sách không có phần tử số hợp lệ nào");
        }

        return (double) sum / validElementsCount;
    }
}
```

### Mã nguồn JUnit 5 Test Case:

```java
import org.junit.jupiter.api.Test;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class AverageCalculatorTest {

    @Test
    void calculateAverage_WithValidList_ShouldReturnCorrectAverage() {
        List<Integer> numbers = Arrays.asList(2, 4, 6, 8);
        double result = AverageCalculator.calculateAverage(numbers);
        assertEquals(5.0, result, "Trung bình của 2,4,6,8 phải là 5.0");
    }

    @Test
    void calculateAverage_WithNullList_ShouldThrowException() {
        Exception exception = assertThrows(IllegalArgumentException.class, () -> {
            AverageCalculator.calculateAverage(null);
        });
        assertEquals("Danh sách không được null hoặc rỗng", exception.getMessage());
    }

    @Test
    void calculateAverage_WithEmptyList_ShouldThrowException() {
        List<Integer> emptyList = Collections.emptyList();
        Exception exception = assertThrows(IllegalArgumentException.class, () -> {
            AverageCalculator.calculateAverage(emptyList);
        });
        assertEquals("Danh sách không được null hoặc rỗng", exception.getMessage());
    }
    
    @Test
    void calculateAverage_WithNullElement_ShouldIgnoreNullAndCalculateCorrectly() {
        // Có phần tử null bên trong list
        List<Integer> numbersWithNull = Arrays.asList(2, null, 8);
        double result = AverageCalculator.calculateAverage(numbersWithNull);
        assertEquals(5.0, result, "Bỏ qua phần tử null, tổng là 10 chia cho 2 phần tử hợp lệ");
    }
}
```
```
