![alt text](image.png)

# How to Use the Supabase API: A Comprehensive Guide

Supabase has quickly emerged as a powerful open-source alternative to Firebase, offering developers a suite of tools built around a PostgreSQL database. At its core, Supabase provides an instant, real-time API layer on top of your database, significantly accelerating backend development. This guide provides a comprehensive overview of how to leverage the Supabase API, covering everything from initial setup and basic operations to security, customization, and type safety.

# **1. Introduction: What is the Supabase API?**

Unlike traditional backend development, where you might spend significant time building REST or GraphQL endpoints to interact with your database, Supabase automatically generates a secure and high-performance API for you. When you create a table in your Supabase PostgreSQL database, Supabase uses **PostgREST**, an open-source tool, to discover your database schema and provide corresponding RESTful endpoints.

# **Key benefits:**

- **Instant backend:** Get functional API endpoints as soon as you define your database schema.
- **Real-time capabilities:** Subscribe to database changes via WebSockets.
- **PostgreSQL-based:** Leverage the power, flexibility, and maturity of PostgreSQL, including features like Row-Level Security (RLS).
- **Multiple interaction methods:** Interact via REST, GraphQL (community-supported), or Supabase client libraries (JavaScript, Python, Dart, etc.).
- **Scalability:** Create custom serverless functions (**Edge Functions**) for complex logic or integration.

This guide focuses primarily on the REST API and its interaction through client libraries, as well as Supabase's Edge Functions.

# **2. Getting started with the Supabase API**

The easiest way to understand the Supabase API is to dive right in. Assuming you have set up a Supabase project (if not, visit supabase.com and create a free one) and created a simple table, for example, **`profiles`**:

```
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

**(Note:** The example above uses **`profiles`**, which aligns with Supabase's standard examples. This concept applies to the **`todos`** table or any other table you create.)

# **Find your API credentials:**

Each Supabase project comes with unique API credentials:

1. **Project URL:** Your unique Supabase endpoint (e.g., **`https://&lt;your-project-ref&gt;.supabase.co`**).
2. **API Keys:** Found in your Supabase Project Dashboard under **`Project Settings`** > **`API`**.
- **`anon`** (guest): This key is safe to use in client-side applications (such as browsers or mobile apps). It relies on Row-Level Security (RLS) to control data access.
- **`service_role`** key: This is a secret key with full administrative privileges, bypassing RLS. **Never disclose this key in client-side code.** Use it only in secure server environments (such as backend servers or serverless functions).

# **Interacting with the API (using the Supabase JS Client Library):**

Supabase provides client libraries to simplify API interactions. Here's how you'll use the JavaScript library (**`supabase-js`**):

```jsx
// 1. Nhập và khởi tạo khách hàng
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
// deleteProfile('some-uuid-v4');

```

This starter table demonstrates basic CRUD (Create, Read, Update, Delete) operations using the client library, which internally calls the REST API.

# **3. Supabase REST API Auto-Generation**

While client libraries are convenient, understanding the underlying REST API generated by PostgREST is essential.

# **API endpoint structure:**

The basic URL for the REST API is typically: **`https://&lt;your-project-ref&gt;.supabase.co/rest/v1/`**

Automatically generated endpoints for your tables:

- **`GET /rest/v1/your_table_name`**: Retrieve rows from the table.
- **`POST /rest/v1/your_table_name`**: Insert new rows into the table.
- **`PATCH /rest/v1/your_table_name`**: Update existing rows in the table.
- **`DELETE /rest/v1/your_table_name`**: Delete rows from the table.

# **Authentication:**

API requests must include your API key in the **`apikey`** header and typically an **`Authorization`** header containing **`Bearer &lt;your-api-key&gt;`** (usually the same **`anon`** key for client-side requests, or the **`service_role`** key for server-side requests).

```visual-basic
apikey: <your-anon-or-service-role-key>
Authorization: Bearer <your-anon-or-service-role-key>

```

# **Common operations (using `curl` as an example):**

Copy the examples below using **`curl`** directly against the REST API. Replace the placeholders as appropriate.

**Retrieve Data (GET):**

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles?select=*' \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>"

```

**Insert Data (POST):**

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles' \
  -X POST \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \# Tùy chọn: Trả về hàng đã chèn \
  -d '{ "id": "some-uuid", "username": "rest_user" }'

```

**Update Data (PATCH):** (Update the profile where the username is 'rest_user')

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles?username=eq.rest_user' \
  -X PATCH \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{ "website": "https://example.com" }'

```

**Delete Data (DELETE):** (Delete the record where the username is 'rest_user')

```livescript
curl 'https://<ref>.supabase.co/rest/v1/profiles?username=eq.rest_user' \
  -X DELETE \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <anon-key>"

```

# **Filter, Select, Sort, Pagination:**

The REST API supports powerful queries through URL parameters:

- **Select Columns:** **`?select=column1,column2`**
- **Filter (Equal):** **`?column_name=eq.value`** (e.g., **`?id=eq.some-uuid`**)
- **Filter (Other Operators):** **`gt`** (greater than), **`lt`** (less than), **`gte`**, **`lte`**, **`neq`** (not equal), **`like`**, **`ilike`** (contains, case-insensitive), **`in`** (e.g., **`?status=in.(active,pending)`**)
- **Sort:** **`?order=column_name.asc`** or **`?order=column_name.desc`** (add **`.nullsfirst`** or **`.nullslast`** if needed)
- **Pagination:** **`?limit=10&offset=0`** (retrieve the first 10), **`?limit=10&offset=10`** (retrieve the next 10)

# **Automatically generated API documentation:**

One of the most useful features of Supabase is the automatically generated API documentation available directly in your project dashboard.

1. Go to your Supabase project.
2. Click on the API Documentation icon (usually represented by **`&lt;&gt;`**) on the left-hand sidebar.
3. Select a table under the "Tables and Views" section.
4. You will see detailed documentation for the specific REST endpoints for that table, including:
- Example requests (**`Bash/curl`**, JavaScript).
- Available filters, selections, and adjustments.
- Column descriptions and data types.

This interactive documentation is invaluable for understanding how to structure your API requests.

# **4. Generate Types to Enhance API Development with Supabase**

For projects using TypeScript or other typed languages, Supabase provides a way to generate type definitions directly from your database schema. This offers significant benefits:

- **Type Safety:** Catch errors at compile time instead of runtime.
- **Autocompletion:** Get smart suggestions in your code editor for table names, column names, and function parameters.
- **Improved Maintenance:** Types act as living documentation for your data structures.

# **Generate types using the Supabase CLI:**

1. **Install Supabase CLI:** Follow the instructions at **`https://supabase.com/docs/guides/cli`**.
2. **Log in:** **`supabase login`**
3. **Link your project:** **`supabase link --project-ref &lt;your-project-ref&gt;`** (Run this in your local project directory). You may need to provide a database password.
4. **Generate types:**

```
supabase gen types typescript --linked > src/database.types.ts
# Hoặc chỉ định project-id nếu không liên kết hoặc ở một ngữ cảnh khác
# supabase gen types typescript --project-id <your-project-ref> > src/database.types.ts

```

This command checks the database schema of your linked Supabase project and exports a TypeScript file (**`database.types.ts`** in this example) containing interfaces for your tables, views, and function parameters/return types.

# **Using the generated types:**

You can import these types into your application code:

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

Generating types is a highly recommended practice for robust application development with Supabase.

# **5. Create Custom Supabase API Routes with Edge Functions**

Although the automatically generated REST API includes standard CRUD operations, you will often need custom server-side logic for:

- Integration with third-party services (e.g., Stripe, Twilio).
- Performing complex calculations or data aggregation.
- Running logic that requires high permissions (**`service_role`** key) without exposing the key to clients.
- Enforcing complex business rules.

Supabase **Edge Functions** provide a way to deploy global Deno-based TypeScript functions at the edge, close to your users.

# **Create an Edge Function:**

1. **Initialize Functions (if not already done):** **`supabase functions new hello-world`** (run in your linked project directory). This creates a file **`supabase/functions/hello-world/index.ts`**.

**Write your function code:**

```
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

**Deploy the function:**

```sql
supabase functions deploy hello-world --no-verify-jwt
# Sử dụng --no-verify-jwt cho các hàm có thể truy cập công cộng
# Bỏ qua nó hoặc đặt --verify-jwt=true để yêu cầu một JWT xác thực Supabase hợp lệ

```

**Call the function:**

You can call deployed functions via HTTP POST requests (or GET, depending on the function logic) to their unique endpoint:

**`https://&lt;your-project-ref&gt;.supabase.co/functions/v1/hello-world`**

Using **`curl`**:

```livescript
curl -X POST 'https://<ref>.supabase.co/functions/v1/hello-world' \
  -H "Authorization: Bearer <user-jwt-if-required>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Functions"}' # Nội dung yêu cầu tùy chọn

```

Or use the Supabase JS client:

```php
const { data, error } = await supabase.functions.invoke('hello-world', {
  method: 'POST', // hoặc 'GET', v.v.
  body: { name: 'Functions' }
})

```

Edge Functions are a powerful tool for extending Supabase's backend capabilities beyond simple database operations.

# **6. API Keys and Supabase API Security**

Understanding API keys and implementing proper security measures is non-negotiable.

# **API Keys Summary:**

- **`anon`** (client): For client-side use. Fully relies on **Row-Level Security (RLS)** to protect data.
- **`service_role`** key: For server-side use only. Bypasses RLS and grants full access to the database. **Protect this key carefully.**

# **The importance of Row-Level Security (RLS):**

**RLS** is the foundation of Supabase security when using the **`anon`** key. It allows you to define sophisticated access control policies directly within the PostgreSQL database. These policies are essentially SQL rules that determine which rows a user can view, insert, update, or delete based on their authentication status, user ID, role, or other criteria.

# **Enabling RLS:**

By default, RLS is **disabled** on new tables. You **must** enable it for any tables you intend to access from the client side using the **`anon`** key.

```
-- Bật RLS trên bảng 'profiles'
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- QUAN TRỌNG: Nếu không có chính sách nào được xác định sau khi bật RLS,
-- quyền truy cập mặc nhiên bị từ chối cho tất cả các thao tác (trừ chủ sở hữu bảng).

```

# **Create RLS Policies:**

Policies define the **`USING`** clause (for read access such as **`SELECT`**, **`UPDATE`**, **`DELETE`**) and the **`WITH CHECK`** clause (for write access such as **`INSERT`**, **`UPDATE`**).

**Example 1: Allow logged-in users to read all records:**

```
CREATE POLICY "Cho phép truy cập đọc đã xác thực"
ON profiles FOR SELECT
USING ( auth.role() = 'authenticated' );

```

**Example 2: Allow users to view only their own records:**

```
CREATE POLICY "Cho phép truy cập đọc cá nhân"
ON profiles FOR SELECT
USING ( auth.uid() = id ); -- Giả sử cột 'id' khớp với UUID của người dùng từ Supabase Auth

```

**Example 3: Allow users to update only their own records:**

```
CREATE POLICY "Cho phép truy cập cập nhật cá nhân"
ON profiles FOR UPDATE
USING ( auth.uid() = id ) -- Xác định những hàng nào có thể nhắm đến để cập nhật
WITH CHECK ( auth.uid() = id ); -- Đảm bảo mọi *dữ liệu mới* vẫn khớp với điều kiện

```

**Example 4: Allow users to insert their own records:**

```
CREATE POLICY "Cho phép truy cập chèn cá nhân"
ON profiles FOR INSERT
WITH CHECK ( auth.uid() = id );

```

You can view, create, and manage RLS policies directly in the Supabase Dashboard under **`Authentication`** > **`Policies`**.

# **Key Security Principles:**

1. **Always enable RLS** on tables accessed via the **`anon`** key.
2. **Define clear policies** for **`SELECT`**, **`INSERT`**, **`UPDATE`**, **`DELETE`** when necessary. Start with restrictive policies and carefully grant access.
3. **Never expose  the`service_role` key** in client-side code or insecure environments.
4. Use **Edge Functions** for operations requiring high privileges or complex server-side logic, and protect your **`service_role`** key in secure function environment variables.
5. Regularly **review your RLS policies** to ensure they meet your application's security requirements.

# **7. Map SQL Concepts to the Supabase API (SQL to API)**

If you are familiar with SQL, understanding how common SQL operations map to the Supabase API (both REST and client libraries) is very useful.

**`SELECT * FROM my_table;`**

- REST: **`GET /rest/v1/my_table?select=*`**
- JS: **`supabase.from('my_table').select('*')`**

**`SELECT column1, column2 FROM my_table WHERE id = 1;`**

- REST: **`GET /rest/v1/my_table?select=column1,column2&id=eq.1`**
- JS: **`supabase.from('my_table').select('column1, column2').eq('id', 1)`**

**`INSERT INTO my_table (column1, column2) VALUES ('value1', 'value2');`**

- REST: **`POST /rest/v1/my_table`** with JSON content **`{"column1": "value1", "column2": "value2"}`**
- JS: **`supabase.from('my_table').insert({ column1: 'value1', column2: 'value2' })`**

**`UPDATE my_table SET column1 = 'new_value' WHERE id = 1;`**

- REST: **`PATCH /rest/v1/my_table?id=eq.1`** with JSON content **`{"column1": "new_value"}`**
- JS: **`supabase.from('my_table').update({ column1: 'new_value' }).eq('id', 1)`**

**`DELETE FROM my_table WHERE id = 1;`**

- REST: **`DELETE /rest/v1/my_table?id=eq.1`**
- JS: **`supabase.from('my_table').delete().eq('id', 1)`**

**Combination:** While direct SQL **`JOIN`** syntax is not used in basic REST calls, you can retrieve related data using:

- **Foreign Key Relationships:** **`?select=*,related_table(*)`** retrieves data from related tables defined by foreign keys.
- JS: **`supabase.from('my_table').select('*, related_table(*)')`**
- **RPC (Remote Procedure Call):** For complex joins or logic, you create a PostgreSQL function and call it via the API.

```
-- Ví dụ hàm SQL
CREATE FUNCTION get_user_posts(user_id uuid)
RETURNS TABLE (post_id int, post_content text) AS $$
  SELECT posts.id, posts.content
  FROM posts
  WHERE posts.author_id = user_id;
$$ LANGUAGE sql;

```

- REST: **`POST /rest/v1/rpc/get_user_posts`** with JSON content **`{"user_id": "some-uuid"}`**
- JS: **`supabase.rpc('get_user_posts', { user_id: 'some-uuid' })`**

# **8. Using Custom Schemas with the Supabase API**

By default, the tables you create in the Supabase SQL Editor are in the **`public`** schema. To organize better, create namespaces, or manage access, you may want to use custom PostgreSQL schemas.

# **Create a Custom Schema:**

```
CREATE SCHEMA private_schema;

```

# **Create a Table in a Custom Schema:**

```
CREATE TABLE private_schema.sensitive_data (
  id serial primary key,
  payload jsonb
);

```

# **Access Tables in Custom Schemas via the API:**

The Supabase PostgREST layer automatically detect tables in schemas other than **`public`**.

- **REST API:** The API endpoints remain the same (**`/rest/v1/table_name`**), but PostgREST automatically displays tables from other schemas. You may need to manage access via roles and permissions in PostgreSQL if you want better schema-level access control beyond standard RLS. If there are name collisions (same table name in **`public`** and another schema), you may need specific configuration or use RPC. Check the PostgREST documentation for handling schema visibility if needed.
- **Client Libraries:** Client libraries work seamlessly. You simply reference table names as usual:

```
// Truy cập một bảng trong 'private_schema' (giả sử RLS/quyền cho phép)
const { data, error } = await supabase
  .from('sensitive_data') // Không cần tiền tố với tên lược đồ ở đây
  .select('*')
  .eq('id', 1);

```

Supabase handles mapping table names to the correct schema behind the scenes. Ensure your RLS policies reference the correct tables if they involve queries or functions across schemas.

Using custom schemas is a standard practice in PostgreSQL that Supabase fully supports, enabling a more structured database organization.

# **9. Conclusion**

The Supabase API provides an extremely efficient way to build applications by offering instant, secure, and scalable access to your PostgreSQL database. From automatically generated REST endpoints and useful client libraries to the robust security provided by Row-Level Security and the scalability offered by Edge Functions, Supabase empowers developers to focus on building features rather than tedious backend infrastructure.

By understanding the core concepts—API keys, RLS, REST structure, type generation, and custom functions—you can effectively leverage the full power of the Supabase platform. Remember to prioritize security.