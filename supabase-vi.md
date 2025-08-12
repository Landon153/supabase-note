![alt text](image.png)

## Supabase

[**Cách Sử Dụng Supabase API: Hướng Dẫn Toàn Diện**](https://www.notion.so/C-ch-S-D-ng-Supabase-API-H-ng-D-n-To-n-Di-n-24d8062abe6480858b08fc6c192f5113?pvs=21)

Supabase đã nhanh chóng trở thành một lựa chọn mã nguồn mở mạnh mẽ thay thế cho Firebase, cung cấp cho các nhà phát triển một bộ công cụ được xây dựng xung quanh cơ sở dữ liệu PostgreSQL. Tại cốt lõi, Supabase cung cấp một lớp API tức thì, thời gian thực trên cơ sở dữ liệu của bạn, tăng tốc đáng kể quá trình phát triển backend. Hướng dẫn này cung cấp cái nhìn tổng quan toàn diện về cách tận dụng API của Supabase, và bao quát mọi thứ từ việc thiết lập ban đầu và các thao tác cơ bản đến bảo mật, tùy chỉnh và sự an toàn kiểu.

💡

Bạn muốn một công cụ kiểm thử API tuyệt vời sinh ra [tài liệu API đẹp mắt](https://apidog.com/api-doc/)?Bạn muốn một nền tảng tích hợp, Tất cả trong một cho Nhóm phát triển của bạn làm việc cùng nhau với [năng suất tối đa](https://apidog.com/api-testing/)?Apidog đáp ứng tất cả yêu cầu của bạn và [thay thế Postman với giá cả phải chăng hơn nhiều](https://apidog.com/compare/apidog-vs-postman/)!

# **1. Giới thiệu: Supabase API là gì?**

Không giống như phát triển backend truyền thống, nơi bạn có thể dành thời gian đáng kể để xây dựng các điểm cuối REST hoặc GraphQL để tương tác với cơ sở dữ liệu của bạn, Supabase tự động tạo ra một API an toàn và hiệu suất cao cho bạn. Khi bạn tạo một bảng trong cơ sở dữ liệu PostgreSQL Supabase của bạn, Supabase sử dụng **PostgREST**, một công cụ mã nguồn mở, để khám phá lược đồ cơ sở dữ liệu của bạn và cung cấp các điểm cuối RESTful tương ứng.

# **Lợi ích chính:**

- **Backend tức thì:** Nhận các điểm cuối API chức năng ngay khi bạn xác định lược đồ cơ sở dữ liệu của mình.
- **Khả năng thời gian thực:** Đăng ký những thay đổi cơ sở dữ liệu qua WebSockets.
- **Dựa trên PostgreSQL:** Tận dụng sức mạnh, tính linh hoạt và độ trưởng thành của PostgreSQL, bao gồm các tính năng như Bảo mật cấp hàng (RLS).
- **Nhiều phương thức tương tác:** Tương tác qua REST, GraphQL (hỗ trợ cộng đồng) hoặc các thư viện khách hàng của Supabase (JavaScript, Python, Dart, v.v.).
- **Tính mở rộng:** Tạo các hàm không máy chủ tùy chỉnh (**Edge Functions**) cho logic phức tạp hoặc tích hợp.

Hướng dẫn này tập trung chủ yếu vào API REST và tương tác của nó thông qua các thư viện khách hàng, cũng như các Hàm Edge của Supabase.

# **2. Bắt đầu với API Supabase**

Cách dễ nhất để hiểu API Supabase là tham gia ngay vào đó. Giả sử bạn đã thiết lập một dự án Supabase (nếu không, hãy truy cập supabase.com và tạo một cái miễn phí) và đã tạo một bảng đơn giản, ví dụ, **`profiles`**:

```sql
-- Tạo một bảng cho các hồ sơ công khai
create table profiles (
  id uuid references auth.users not null primary key,
  updated_at timestamp with time zone,
  username text unique,
  avatar_url text,
  website text,

  constraint username_length check (char_length(username) >= 3)
);

-- Thiết lập Bảo mật cấp hàng (RLS)
-- Xem https://supabase.com/docs/guides/auth/row-level-security để biết thêm chi tiết.
alter table profiles
  enable row level security;

create policy "Hồ sơ công khai có thể được xem bởi mọi người." on profiles
  for select using (true);

create policy "Người dùng có thể chèn hồ sơ của chính họ." on profiles
  for insert with check (auth.uid() = id);

create policy "Người dùng có thể cập nhật hồ sơ của chính mình." on profiles
  for update using (auth.uid() = id);

-- Trigger này tự động tạo một mục hồ sơ khi một người dùng mới đăng ký qua Supabase Auth.
-- Xem https://supabase.com/docs/guides/auth/managing-user-data#using-triggers để biết thêm chi tiết.
create function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, username)
  values (new.id, new.raw_user_meta_data->>'username');
  return new;
end;
$$ language plpgsql security definer;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

```

**(Lưu ý:** Ví dụ trên sử dụng **`profiles`**, phù hợp với các ví dụ tiêu chuẩn của Supabase. Khái niệm này áp dụng cho bảng **`todos`** hoặc bất kỳ bảng nào khác bạn tạo.)

# **Tìm thông tin xác thực API của bạn:**

Mỗi dự án Supabase đi kèm với các thông tin xác thực API độc đáo:

1. **URL Dự án:** Điểm kết Supabase độc nhất của bạn (ví dụ, **`https://<your-project-ref>.supabase.co`**).
2. **API Keys:** Được tìm thấy trong Bảng điều khiển Dự án Supabase của bạn dưới **`Cài đặt Dự án`** > **`API`**.

- **`anon`** (khách): Khóa này an toàn để sử dụng trong các ứng dụng phía khách (như trình duyệt hoặc ứng dụng di động). Nó phụ thuộc vào Bảo mật cấp hàng (RLS) để kiểm soát việc truy cập dữ liệu.
- **`service_role`** key: Đây là một khóa bí mật với quyền quản trị đầy đủ, bỏ qua RLS. **Không bao giờ tiết lộ khóa này trong mã phía khách.** Sử dụng nó chỉ trong môi trường máy chủ an toàn (như máy chủ backend hoặc các hàm không máy chủ).

# **Tương tác với API (sử dụng Thư viện Khách hàng Supabase JS):**

Supabase cung cấp các thư viện khách hàng để đơn giản hóa các tương tác API. Đây là cách bạn sẽ sử dụng thư viện JavaScript (**`supabase-js`**):

```jsx
**// 1. Nhập và khởi tạo khách hàng
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = 'https://<your-project-ref>.supabase.co'
const supabaseAnonKey = '<your-anon-key>'

const supabase = createClient(supabaseUrl, supabaseAnonKey)

// 2. Lấy dữ liệu (SELECT *)
async function getProfiles() {
  const { data, error } = await supabase
    .from('profiles')
    .select('*')

  if (error) console.error('Lỗi khi lấy hồ sơ:', error)
  else console.log('Hồ sơ:', data)
}

// 3. Chèn dữ liệu (INSERT)
async function createProfile(userId, username) {
  const { data, error } = await supabase
    .from('profiles')
    .insert([
      { id: userId, username: username, updated_at: new Date() },
    ])
    .select() // Trả về dữ liệu đã chèn

  if (error) console.error('Lỗi khi tạo hồ sơ:', error)
  else console.log('Hồ sơ đã tạo:', data)
}

// 4. Cập nhật dữ liệu (UPDATE)
async function updateProfileUsername(userId, newUsername) {
  const { data, error } = await supabase
    .from('profiles')
    .update({ username: newUsername, updated_at: new Date() })
    .eq('id', userId) // Chỉ cập nhật nơi id khớp
    .select()

  if (error) console.error('Lỗi khi cập nhật hồ sơ:', error)
  else console.log('Hồ sơ đã cập nhật:', data)
}

// 5. Xóa dữ liệu (DELETE)
async function deleteProfile(userId) {
  const { data, error } = await supabase
    .from('profiles')
    .delete()
    .eq('id', userId) // Chỉ xóa nơi id khớp

  if (error) console.error('Lỗi khi xóa hồ sơ:', error)
  // Lưu ý: Xóa thường trả về dữ liệu tối thiểu về thành công trừ khi .select() được sử dụng *trước* .delete() trên một số phiên bản/cài đặt.
  else console.log('Hồ sơ đã xóa thành công')
}

// Ví dụ sử dụng (giả sử bạn có ID người dùng)
// getProfiles();
// createProfile('some-uuid-v4', 'new_user');
// updateProfileUsername('some-uuid-v4', 'updated_username');
// deleteProfile('some-uuid-v4');**

```

Bảng khởi động này minh họa các thao tác CRUD (Tạo, Đọc, Cập nhật, Xóa) cơ bản bằng cách sử dụng thư viện khách hàng, mà nội bộ gọi API REST.

# **3. Supabase REST API Tự động sinh**

Mặc dù các thư viện khách hàng rất tiện lợi, nhưng hiểu API REST bên dưới được tạo bởi PostgREST là điều thiết yếu.

# **Cấu trúc Điểm kết API:**

URL cơ bản cho API REST thường là: **`https://<your-project-ref>.supabase.co/rest/v1/`**

Các điểm kết tự động được tạo cho các bảng của bạn:

- **`GET /rest/v1/your_table_name`**: Lấy các hàng từ bảng.
- **`POST /rest/v1/your_table_name`**: Chèn các hàng mới vào bảng.
- **`PATCH /rest/v1/your_table_name`**: Cập nhật các hàng hiện có trong bảng.
- **`DELETE /rest/v1/your_table_name`**: Xóa các hàng từ bảng.

# **Xác thực:**

Các yêu cầu API phải bao gồm khóa API của bạn trong header **`apikey`** và thường là một header **`Authorization`** chứa **`Bearer <your-api-key>`** (thường là cùng một **`anon`** key cho các yêu cầu phía khách, hoặc **`service_role`** key cho phía máy chủ).

```visual-basic
apikey: <your-anon-or-service-role-key>
Authorization: Bearer <your-anon-or-service-role-key>

```

# **Các thao tác phổ biến (sử dụng `curl` ví dụ):**

Hãy sao chép các ví dụ trước bằng cách sử dụng **`curl`** trực tiếp chống lại API REST. Thay thế các placeholder cho phù hợp.

**Lấy Dữ liệu (GET):**

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles?select=*' \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>"

```

**Chèn Dữ liệu (POST):**

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles' \
  -X POST \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \# Tùy chọn: Trả về hàng đã chèn \
  -d '{ "id": "some-uuid", "username": "rest_user" }'

```

**Cập nhật Dữ liệu (PATCH):** (Cập nhật hồ sơ nơi username là 'rest_user')

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles?username=eq.rest_user' \
  -X PATCH \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{ "website": "https://example.com" }'

```

**Xóa Dữ liệu (DELETE):** (Xóa hồ sơ nơi username là 'rest_user')

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles?username=eq.rest_user' \
  -X DELETE \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>"

```

# **Lọc, Chọn, Sắp xếp, Phân trang:**

API REST hỗ trợ truy vấn mạnh mẽ thông qua các tham số URL:

- **Chọn Cột:** **`?select=column1,column2`**
- **Lọc (Bằng nhau):** **`?column_name=eq.value`** (ví dụ **`?id=eq.some-uuid`**)
- **Lọc (Các Toán tử Khác):** **`gt`** (lớn hơn), **`lt`** (nhỏ hơn), **`gte`**, **`lte`**, **`neq`** (không bằng), **`like`**, **`ilike`** (chứa không phân biệt hoa thường), **`in`** (ví dụ, **`?status=in.(active,pending)`**)
- **Sắp xếp:** **`?order=column_name.asc`** hoặc **`?order=column_name.desc`** (thêm **`.nullsfirst`** hoặc **`.nullslast`** nếu cần)
- **Phân trang:** **`?limit=10&offset=0`** (lấy 10 đầu tiên), **`?limit=10&offset=10`** (lấy 10 tiếp theo)

# **Tài liệu API Tự động sinh:**

Một trong những tính năng hữu ích nhất của Supabase là tài liệu API tự động sinh có sẵn ngay trong bảng điều khiển dự án của bạn.

1. Chuyển đến dự án Supabase của bạn.
2. Nhấp vào biểu tượng Tài liệu API (thường có dạng **`<>`**) ở thanh bên trái.
3. Chọn một bảng dưới mục "Bảng và Góc nhìn".
4. Bạn sẽ thấy tài liệu chi tiết cho các điểm cuối REST cụ thể cho bảng đó, bao gồm:

- Các yêu cầu ví dụ (Bash/**`curl`**, JavaScript).
- Các bộ lọc, lựa chọn và điều chỉnh có sẵn.
- Mô tả của các cột và kiểu dữ liệu.

Tài liệu tương tác này rất quý giá để hiểu cách cấu trúc các yêu cầu API của bạn.

# **4. Sinh ra các Kiểu để Nâng cao Phát triển Sử dụng API Supabase**

Đối với các dự án sử dụng TypeScript hoặc các ngôn ngữ có kiểu khác, Supabase cung cấp một cách để sinh ra định nghĩa kiểu trực tiếp từ lược đồ cơ sở dữ liệu của bạn. Điều này mang lại những lợi ích đáng kể:

- **An toàn Kiểu:** Bắt lỗi vào thời gian biên dịch thay vì thời gian thực thi.
- **Hoàn thành Tự động:** Nhận gợi ý thông minh trong trình soạn thảo mã của bạn cho các tên bảng, tên cột và tham số hàm.
- **Duy trì Cải thiện:** Các kiểu hoạt động như tài liệu sống cho các cấu trúc dữ liệu của bạn.

# **Sinh ra Các kiểu bằng cách sử dụng Supabase CLI:**

1. **Cài đặt Supabase CLI:** Thực hiện theo các hướng dẫn tại **`https://supabase.com/docs/guides/cli`**.
2. **Đăng nhập:** **`supabase login`**
3. **Liên kết dự án của bạn:** **`supabase link --project-ref <your-project-ref>`** (Chạy điều này trong thư mục dự án cục bộ của bạn). Bạn có thể cần cung cấp một mật khẩu cơ sở dữ liệu.
4. **Xin sinh ra các kiểu:**

```
supabase gen types typescript --linked > src/database.types.ts
# Hoặc chỉ định project-id nếu không liên kết hoặc ở một ngữ cảnh khác
# supabase gen types typescript --project-id <your-project-ref> > src/database.types.ts

```

Lệnh này kiểm tra lược đồ cơ sở dữ liệu của dự án Supabase đã liên kết của bạn và xuất một tệp TypeScript (**`database.types.ts`** trong ví dụ này) chứa các interface cho các bảng, góc nhìn và các tham số/kiểu trả về của hàm của bạn.

# **Sử dụng các Kiểu đã sinh:**

Bạn có thể nhập các kiểu này vào mã ứng dụng của mình:

```jsx
import { createClient } from '@supabase/supabase-js'
// Nhập các kiểu đã sinh
import { Database } from './database.types' // Điều chỉnh đường dẫn nếu cần

const supabaseUrl = 'https://<your-project-ref>.supabase.co'
const supabaseAnonKey = '<your-anon-key>'

// Cung cấp kiểu Database để createClient
const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey)

// Bây giờ bạn nhận được an toàn kiểu và hoàn thành tự động!
async function getSpecificUserProfile(username: string) {
  // Hoàn thành tự động cho tên bảng ('profiles')
  const { data, error } = await supabase
    .from('profiles')
    // Hoàn thành tự động cho tên cột ('id', 'username', 'website')
    .select('id, username, website')
    // Kiểm tra kiểu giá trị dựa trên kiểu cột
    .eq('username', username)
    .single() // Mong đợi một kết quả duy nhất hoặc null

  if (error) {
    console.error('Lỗi khi lấy hồ sơ:', error)
    return null;
  }

  // 'data' bây giờ được kiểu đúng dựa trên truy vấn chọn của bạn
  if (data) {
    console.log(`ID người dùng: ${data.id}, Trang web: ${data.website}`);
    // console.log(data.non_existent_column); // <-- Lỗi TypeScript!
  }

  return data;
}

```

Việc sinh ra các kiểu là một thực hành rất được khuyến nghị cho phát triển ứng dụng mạnh mẽ với Supabase.

# **5. Tạo các Route API Supabase Tùy Chỉnh với Các Hàm Edge**

Mặc dù API REST tự động sinh bao gồm các thao tác CRUD tiêu chuẩn, bạn sẽ thường cần logic phía máy chủ tùy chỉnh cho:

- Tích hợp với các dịch vụ bên thứ ba (ví dụ như Stripe, Twilio).
- Thực hiện các phép toán phức tạp hoặc tổng hợp dữ liệu.
- Chạy logic yêu cầu quyền hạn cao (**`service_role`** key) mà không tiết lộ khóa cho khách hàng.
- Thực thi các quy tắc kinh doanh phức tạp.

Supabase **Edge Functions** cung cấp một cách để triển khai các hàm TypeScript dựa trên Deno toàn cầu tại biên, gần người dùng của bạn.

# **Tạo một Hàm Edge:**

1. **Khởi tạo Các Hàm (nếu chưa thực hiện):** **`supabase functions new hello-world`** (chạy trong thư mục dự án đã liên kết của bạn). Điều này tạo ra một tệp **`supabase/functions/hello-world/index.ts`**.

**Viết mã hàm của bạn:**

```jsx
// supabase/functions/hello-world/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts' // Sử dụng phiên bản std thích hợp

serve(async (req) => {
  // Bạn có thể truy cập tiêu đề yêu cầu, phương thức, nội dung, v.v. từ 'req'
  console.log(`Yêu cầu đã nhận cho: ${req.url}`);

  // Ví dụ: Truy cập DB Supabase từ trong hàm
  // Lưu ý: Cần thiết lập khách hàng Supabase *bên trong* hàm
  // Sử dụng biến môi trường cho các bí mật!
  // import { createClient } from '@supabase/supabase-js'
  // const supabaseAdmin = createClient(
  //   Deno.env.get('SUPABASE_URL') ?? '',
  //   Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
  // )
  // const { data: users, error } = await supabaseAdmin.from('profiles').select('*').limit(10);

  const data = {
    message: `Xin chào từ Edge!`,
    // users: users // Ví dụ nếu lấy dữ liệu
  }

  return new Response(
    JSON.stringify(data),
    { headers: { 'Content-Type': 'application/json' } },
  )
})

```

**Triển khai hàm:**

```sql
supabase functions deploy hello-world --no-verify-jwt
# Sử dụng --no-verify-jwt cho các hàm có thể truy cập công cộng
# Bỏ qua nó hoặc đặt --verify-jwt=true để yêu cầu một JWT xác thực Supabase hợp lệ

```

**Gọi hàm:**

Bạn có thể gọi các hàm đã triển khai qua các yêu cầu HTTP POST (hoặc GET, tùy thuộc vào logic hàm) đến điểm kết duy nhất của chúng:

**`https://<your-project-ref>.supabase.co/functions/v1/hello-world`**

Sử dụng **`curl`**:

```livescript
curl -X POST 'https://<ref>.supabase.co/functions/v1/hello-world' \
  -H "Authorization: Bearer <user-jwt-if-required>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Functions"}' # Nội dung yêu cầu tùy chọn

```

Hoặc sử dụng khách hàng Supabase JS:

```php
const { data, error } = await supabase.functions.invoke('hello-world', {
  method: 'POST', // hoặc 'GET', v.v.
  body: { name: 'Functions' }
})

```

Các Hàm Edge là một công cụ mạnh mẽ để mở rộng khả năng backend của Supabase vượt ra ngoài những thao tác cơ sở dữ liệu đơn giản.

# **6. Các API Key và Bảo mật API Supabase của bạn**

Hiểu về API keys và triển khai các biện pháp bảo mật đúng cách là điều không thể thương lượng.

# **Tóm tắt API Keys:**

- **`anon`** (khách): Dành cho việc sử dụng phía khách. Hoàn toàn dựa vào **Bảo mật Cấp hàng (RLS)** để bảo vệ dữ liệu.
- **`service_role`** key: Chỉ dành cho sử dụng phía máy chủ. Bỏ qua RLS, cấp quyền truy cập đầy đủ vào cơ sở dữ liệu. **Bảo vệ khóa này cẩn thận.**

# **Vai trò quan trọng của Bảo mật Cấp hàng (RLS):**

**RLS** là nền tảng của sự bảo mật Supabase khi sử dụng khóa **`anon`**. Nó cho phép bạn xác định các chính sách kiểm soát truy cập tinh vi ngay trong cơ sở dữ liệu PostgreSQL. Các chính sách về cơ bản là các quy tắc SQL xác định những hàng nào mà một người dùng có thể xem, chèn, cập nhật hoặc xóa dựa trên trạng thái xác thực của họ, ID người dùng, vai trò hoặc các tiêu chí khác.

# **Bật RLS:**

Mặc định, RLS là **tắt** trên các bảng mới. Bạn **phải** bật nó cho bất kỳ bảng nào bạn có ý định truy cập từ phía khách sử dụng khóa **`anon`**.

```sql
-- Bật RLS trên bảng 'profiles'
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- QUAN TRỌNG: Nếu không có chính sách nào được xác định sau khi bật RLS,
-- quyền truy cập mặc nhiên bị từ chối cho tất cả các thao tác (trừ chủ sở hữu bảng).

```

# **Tạo Chính sách RLS:**

Các chính sách xác định mệnh đề **`USING`** (cho quyền truy cập đọc như **`SELECT`**, **`UPDATE`**, **`DELETE`**) và mệnh đề **`WITH CHECK`** (cho quyền truy cập ghi như **`INSERT`**, **`UPDATE`**).

**Ví dụ 1: Cho phép người dùng đã đăng nhập đọc tất cả hồ sơ:**

```sql
CREATE POLICY "Cho phép truy cập đọc đã xác thực"
ON profiles FOR SELECT
USING ( auth.role() = 'authenticated' );

```

**Ví dụ 2: Cho phép người dùng chỉ xem hồ sơ của chính họ:**

```sql
CREATE POLICY "Cho phép truy cập đọc cá nhân"
ON profiles FOR SELECT
USING ( auth.uid() = id ); -- Giả sử cột 'id' khớp với UUID của người dùng từ Supabase Auth

```

**Ví dụ 3: Cho phép người dùng cập nhật chỉ hồ sơ của chính họ:**

```sql
CREATE POLICY "Cho phép truy cập cập nhật cá nhân"
ON profiles FOR UPDATE
USING ( auth.uid() = id ) -- Xác định những hàng nào có thể nhắm đến để cập nhật
WITH CHECK ( auth.uid() = id ); -- Đảm bảo mọi *dữ liệu mới* vẫn khớp với điều kiện

```

**Ví dụ 4: Cho phép người dùng chèn hồ sơ của chính họ:**

```sql
CREATE POLICY "Cho phép truy cập chèn cá nhân"
ON profiles FOR INSERT
WITH CHECK ( auth.uid() = id );

```

Bạn có thể xem, tạo và quản lý các chính sách RLS trực tiếp trong Bảng điều khiển Supabase dưới **`Xác thực`** > **`Chính sách`**.

# **Các Nguyên tắc Bảo mật Chính:**

1. **Luôn bật RLS** trên các bảng được truy cập qua khóa **`anon`**.
2. **Xác định các chính sách rõ ràng** cho **`SELECT`**, **`INSERT`**, **`UPDATE`**, **`DELETE`** khi cần. Bắt đầu với các chính sách hạn chế và mở quyền truy cập một cách cẩn thận.
3. **Không bao giờ tiết lộ `service_role` key** trong mã phía khách hoặc môi trường không an toàn.
4. Sử dụng **Các Hàm Edge** cho các thao tác yêu cầu quyền hạn cao hoặc logic phía máy chủ phức tạp, bảo vệ khóa **`service_role`** của bạn trong các biến môi trường an toàn của hàm.
5. Thường xuyên **xem xét các chính sách RLS của bạn** để đảm bảo chúng đáp ứng yêu cầu bảo mật của ứng dụng của bạn.

# **7. Ánh xạ Các Khái niệm SQL tới API Supabase (SQL đến API)**

Nếu bạn quen thuộc với SQL, hiểu cách các thao tác SQL thông thường ánh xạ đến API Supabase (cả REST và thư viện khách hàng) là rất hữu ích.

**`SELECT * FROM my_table;`**

- REST: **`GET /rest/v1/my_table?select=*`**
- JS: **`supabase.from('my_table').select('*')`**

**`SELECT column1, column2 FROM my_table WHERE id = 1;`**

- REST: **`GET /rest/v1/my_table?select=column1,column2&id=eq.1`**
- JS: **`supabase.from('my_table').select('column1, column2').eq('id', 1)`**

**`INSERT INTO my_table (column1, column2) VALUES ('value1', 'value2');`**

- REST: **`POST /rest/v1/my_table`** với nội dung JSON **`{"column1": "value1", "column2": "value2"}`**
- JS: **`supabase.from('my_table').insert({ column1: 'value1', column2: 'value2' })`**

**`UPDATE my_table SET column1 = 'new_value' WHERE id = 1;`**

- REST: **`PATCH /rest/v1/my_table?id=eq.1`** với nội dung JSON **`{"column1": "new_value"}`**
- JS: **`supabase.from('my_table').update({ column1: 'new_value' }).eq('id', 1)`**

**`DELETE FROM my_table WHERE id = 1;`**

- REST: **`DELETE /rest/v1/my_table?id=eq.1`**
- JS: **`supabase.from('my_table').delete().eq('id', 1)`**

**Kết hợp:** Trong khi cú pháp SQL **`JOIN`** trực tiếp không được sử dụng trong các cuộc gọi REST cơ bản, bạn có thể lấy dữ liệu liên quan bằng cách sử dụng:

- **Mối quan hệ Khóa Ngoại:** **`?select=*,related_table(*)`** lấy dữ liệu từ các bảng liên quan được xác định bởi các khóa ngoại.
- JS: **`supabase.from('my_table').select('*, related_table(*)')`**
- **RPC (Cuộc gọi thủ tục từ xa):** Đối với các kết hợp hoặc logic phức tạp, bạn tạo một hàm PostgreSQL và gọi nó qua API.

```
-- Ví dụ hàm SQL
CREATE FUNCTION get_user_posts(user_id uuid)
RETURNS TABLE (post_id int, post_content text) AS $$
  SELECT posts.id, posts.content
  FROM posts
  WHERE posts.author_id = user_id;
$$ LANGUAGE sql;

```

- REST: **`POST /rest/v1/rpc/get_user_posts`** với nội dung JSON **`{"user_id": "some-uuid"}`**
- JS: **`supabase.rpc('get_user_posts', { user_id: 'some-uuid' })`**

# **8. Sử dụng Các Lược đồ Tùy chỉnh với API Supabase**

Mặc định, các bảng bạn tạo trong Trình soạn thảo SQL Supabase nằm trong lược đồ **`public`**. Để tổ chức tốt hơn, tạo không gian tên hoặc quản lý quyền truy cập, bạn có thể muốn sử dụng các lược đồ PostgreSQL tùy chỉnh.

# **Tạo một Lược đồ Tùy chỉnh:**

```sql
CREATE SCHEMA private_schema;

```

# **Tạo Bảng trong một Lược đồ Tùy chỉnh:**

```sql
CREATE TABLE private_schema.sensitive_data (
  id serial primary key,
  payload jsonb
);

```

# **Truy cập các Bảng trong Các Lược đồ Tùy chỉnh thông qua API:**

Tầng PostgREST của Supabase tự động phát hiện các bảng trong các lược đồ khác ngoài **`public`**.

- **REST API:** Các điểm kết API vẫn giống nhau (**`/rest/v1/table_name`**), nhưng PostgREST tự động hiển thị các bảng từ các lược đồ khác. Bạn có thể cần quản lý quyền truy cập thông qua các vai trò và quyền trong PostgreSQL nếu bạn muốn kiểm soát truy cập cấp lược đồ tốt hơn ngoài RLS tiêu chuẩn. Nếu có sự trùng tên (cùng một tên bảng trong **`public`** và một lược đồ khác), bạn có thể cần cấu hình cụ thể hoặc sử dụng RPC. Kiểm tra tài liệu PostgREST để xử lý khả năng hiển thị lược đồ nếu cần.
- **Các Thư viện Khách hàng:** Các thư viện khách hàng hoạt động liền mạch. Bạn chỉ cần tham chiếu đến tên bảng như thường lệ:

```jsx
// Truy cập một bảng trong 'private_schema' (giả sử RLS/quyền cho phép)
const { data, error } = await supabase
  .from('sensitive_data') // Không cần tiền tố với tên lược đồ ở đây
  .select('*')
  .eq('id', 1);

```

Supabase xử lý ánh xạ tên bảng tới lược đồ đúng ở phía sau. Đảm bảo các chính sách RLS của bạn tham chiếu đúng các bảng nếu chúng liên quan đến các truy vấn hoặc hàm giữa các lược đồ.

Sử dụng các lược đồ tùy chỉnh là một thực hành tiêu chuẩn của PostgreSQL mà Supabase hoàn toàn hỗ trợ, cho phép tổ chức cơ sở dữ liệu có cấu trúc hơn.

# **9. Kết luận**

API Supabase cung cấp một cách cực kỳ hiệu quả để xây dựng các ứng dụng bằng cách cung cấp quyền truy cập tức thì, an toàn và có thể mở rộng đến cơ sở dữ liệu PostgreSQL của bạn. Từ các điểm cuối REST tự động sinh và các thư viện khách hàng hữu ích đến sự bảo mật mạnh mẽ được cung cấp bởi Bảo mật Cấp hàng và khả năng mở rộng mà các Hàm Edge cung cấp, Supabase trao quyền cho các nhà phát triển để tập trung vào việc xây dựng các tính năng thay vì hạ tầng backend tẻ nhạt.

Bằng cách hiểu các khái niệm cốt lõi - khóa API, RLS, cấu trúc REST, sinh kiểu và các hàm tùy chỉnh - bạn có thể tận dụng hiệu quả sức mạnh toàn diện của nền tảng Supabase. Hãy nhớ ưu tiên bảo mật
