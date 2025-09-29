# 🎯 Addressables – Phân tích vấn đề & Giải pháp

## ❓ Vấn đề
- Có một **ScriptableObject (SO)** được sử dụng để:
  1. Lưu trữ dữ liệu cấu hình (config data).  
  2. Chứa một `List` được khởi tạo ở runtime, thông qua tham chiếu từ một GameObject trong Scene.  

- Cấu trúc trong project:
  - **Prefab** nằm trong Addressables.  
  - **Scene** không nằm trong Addressables.  
  - ScriptableObject này được tham chiếu ở cả **Prefab** và **Scene**.  

- Hiện tượng:
  - Trên **Editor**: hoạt động bình thường (SO là một instance duy nhất).  
  - Trên **Device build**:  
    - Config data trong SO vẫn tồn tại.  
    - Nhưng `List` trong SO lại bị `null` khi load từ Prefab (mặc dù ở Scene đã init thành công).  

---

## 🔍 Nguyên nhân
- **Editor**: Scene và Prefab đều tham chiếu trực tiếp tới file `.asset` → chỉ có **một bản SO duy nhất**.  
- **Build**:
  - Prefab (Addressables) và Scene (non-Addressables) được đóng gói bằng **hai pipeline khác nhau**.  
  - Mỗi pipeline tự copy ScriptableObject làm dependency → tạo ra **hai instance SO khác nhau**.  
  - Kết quả: SO trong Scene được init, nhưng Prefab lại trỏ sang bản SO khác (chưa init).  

---

## 🛠 Giải pháp
- Đưa **Scene vào Addressables**, để Prefab và Scene cùng đi qua **chung một build pipeline**.  
- Chạy **Analyze → Check Duplicate Asset Dependencies** để phát hiện duplicate.  
- Di chuyển ScriptableObject sang **một group chung**.  
- Build lại → lúc runtime, Prefab và Scene tham chiếu tới **cùng một instance SO duy nhất**.  
- Kết quả: `List` trong SO được giữ nguyên sau khi init, không còn null.  

---

## 📊 Sơ đồ minh họa

```mermaid
flowchart TD
    subgraph Editor
        SO1[ScriptableObject.asset]
        Prefab[Prefab]
        Scene[Scene]
        Prefab --> SO1
        Scene --> SO1
    end

    subgraph Build_Ban_đầu[Build ban đầu]
        PrefabB[Prefab Bundle]
        SceneB[Scene Build]
        SO2[SO Copy A]
        SO3[SO Copy B]
        PrefabB --> SO2
        SceneB --> SO3
    end

    subgraph Build_Sau_khi_fix[Build sau khi đưa Scene vào Addressables]
        PrefabBA[Prefab Bundle]
        SceneBA[Scene Bundle]
        SO_Shared[SO Shared]
        PrefabBA --> SO_Shared
        SceneBA --> SO_Shared
    end

    Editor --> Build_Ban_đầu
    Build_Ban_đầu --> Build_Sau_khi_fix
