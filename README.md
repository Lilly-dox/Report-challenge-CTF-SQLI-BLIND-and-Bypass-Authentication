# Report-challenge-CTF-SQLI-BLIND-and-Bypass-Authentication
# My Flask App

### Mô tả ngắn về challenge :

Challenge bao gồm 3 chức năng chính là login, resetpassword, search.
Và Database với phần password ở dạng hashed sha256

### Ý tưởng sơ bộ triển khai :

Player sẽ khai thác database qua lỗi SQL Blind ở chức năng Search. Lúc này player sẽ có được username. Và phải chuyển sang bypass authen để reset password mới được tạo ra từ seed . Phần này thì sẽ cho phép whitebox code chức năng reset để player có thể lợi dụng code tạo password mới để chạy script tạo password mới sau reset để login lấy flag .

### Mô tả cụ thể về từng tính năng:

- Login :

```jsx
@app.route("/login", methods=["POST"])
def login():
    user = request.form.get("user")
    pwd = request.form.get("pass")
    # Hash the password input
    hashed_password_input = hash_password(pwd)
    # Check if the username matches the regex pattern for @gmail.com.vn
    regex_pattern = r"^[a-zA-Z0-9]+@gmail\.com\.vn$"
    if not re.match(regex_pattern, user):
        # Username does not match the pattern
        return render_template(
            "index.html",
            data="Invalid username format. Please enter a valid username (e.g., example@gmail.com.vn).",
        )

    # Use regex to match username
    regex = r"^" + re.escape(user.split("@")[0]) + r"$"
    # regex = re.escape(user.split("@")[0])
    query = "SELECT * FROM users WHERE username REGEXP %s AND password = %s"
cur = conn.cursor()
    cur.execute(query, (regex, hashed_password_input))
    user_data = cur.fetchone()

    if user_data:
        hashed_password_db = user_data[
            1
        ]  # Assuming hashed password is stored in the second column
        if hashed_password_input == hashed_password_db:
            if str(user_data[0]) == "administratorkjkljn3er7jdshj":
                with open("flag.txt", "r") as f:
                    flag = f.read().strip()
                return f"""
                <center>
                <h1> WELCOME ADMIN </h1>
                <br><br>
                <h3>
                You know Admin has higher privileges ! so in this application, admin is given access to the remote sql editor !
                <br><br>
                !! CLICK THE BELOW BUTTON !!
                </h3>
                <h2>Flag: {flag}</h2>
                </center>
                """
            else:
                return "Hello " + str(user_data[0])
        else:
            return render_template("index.html", data="Invalid Login/Password !!")
    else:
        return render_template("index.html", data="Invalid Login/Password !!")
```

Chức năng login nhận input username , password từ người dùng . Username có regex  ^[a-zA-Z0-9]+@gmail\.com\.vn$ 

. Còn phần input password của người dùng thì sẽ được hash rồi so sánh với hashed_password đã    lưu trong database .

- Reset Password :

 Chức năng reset password thì sẽ reset password theo seed là username và mật khẩu sẽ luôn chỉ có một sau mỗi lần reset . Tức là sau khi player lấy được username , player chỉ cần nhập username vào resetpassword rồi reset . Sau đó, chạy script gen password cố định dựa trên username lấy pass . Rồi login lấy flag .

Đoạn code gen newpassword:

```jsx
if existing_user:
        random.seed(existing_user[0])  # Use username as seed
        new_password = "".join(
            random.choices(
                "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789", k=8
            )
        )
        # Hash the new password
        hashed_password = hash_password(new_password)

        query = "UPDATE users SET password = %s WHERE username = %s"
        cur.execute(query, (hashed_password, user))
        conn.commit()
```

- Search : Chứa lỗ hổng SQL Blind để khai thác database .
ví dụ payload test sqli :

```jsx
' OR EXISTS(SELECT 1 FROM users WHERE username='admin' AND SLEEP(8))--
```

<chắc chắn rằng bạn có khoảng trắng sau -- vì nếu không có khoảng trắng, MySQL sẽ không nhận diện đó là một chú thích và sẽ cố gắng chạy đoạn code phía sau như một phần của truy vấn.>

- Database :

![image](https://github.com/Lilly-dox/Report-challenge-CTF-SQLI-BLIND-and-Bypass-Authentication/assets/130746941/c336ede2-f6b3-4934-ab05-8a5227fa1f1c)

|admin

| admin1234

| administratorkjkljn3er7jdshj
database hiện có 3 account trong bảng users . 

Chỉ account 

```jsx
username = administratorkjkljn3er7jdshj và password = L5dOmUJo
```

Sau khi đăng nhập vào sẽ hiện flag (mật khẩu này là mật khẩu đã được thay đổi sau khi reset trong quá trình test chức năng ) 

### Để triển khai ứng dụng, các bước sau:

Tải file docker-compose.yml
Mở terminal và trỏ đến vị trí thư mục chứa file docker-compose đã tải và lên chạy command: docker-compose up -d
Rồi sau đó truy cập đến 'http://localhost:5000/' để xem app
