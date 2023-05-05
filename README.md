# gitlab
Crack gitlab ee

```
yum install -y curl policycoreutils-python openssh-server
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
yum install -y gitlab-ee
```
Для версии Ruby требуется версия 2.3 или выше
```
yum install -y ruby
```
```
gem install gitlab-license
```
Создаем лицензию

```
require "openssl"
require "gitlab/license"

key_pair = OpenSSL::PKey::RSA.generate(2048)
File.open("license_key", "w") { |f| f.write(key_pair.to_pem) }

public_key = key_pair.public_key
File.open("license_key.pub", "w") { |f| f.write(public_key.to_pem) }

private_key = OpenSSL::PKey::RSA.new File.read("license_key")
Gitlab::License.encryption_key = private_key

license = Gitlab::License.new
license.licensee = {
  "Name" => "none",
  "Company" => "none",
  "Email" => "example@test.com",
}
license.starts_at = Date.new(2020, 1, 1) # 开始时间
license.expires_at = Date.new(2050, 1, 1) # 结束时间
license.notify_admins_at = Date.new(2049, 12, 1)
license.notify_users_at = Date.new(2049, 12, 1)
license.block_changes_at = Date.new(2050, 1, 1)
license.restrictions = {
  active_user_count: 10000,
}

puts "License:"
puts license

data = license.export
puts "Exported license:"
puts data
File.open("GitLabBV.gitlab-license", "w") { |f| f.write(data) }

public_key = OpenSSL::PKey::RSA.new File.read("license_key.pub")
Gitlab::License.encryption_key = public_key

data = File.read("GitLabBV.gitlab-license")
$license = Gitlab::License.import(data)

puts "Imported license:"
puts $license

unless $license
  raise "The license is invalid."
end

if $license.restricted?(:active_user_count)
  active_user_count = 10000
  if active_user_count > $license.restrictions[:active_user_count]
    raise "The active user count exceeds the allowed amount!"
  end
end

if $license.notify_admins?
  puts "The license is due to expire on #{$license.expires_at}."
end

if $license.notify_users?
  puts "The license is due to expire on #{$license.expires_at}."
end

module Gitlab
  class GitAccess
    def check(cmd, changes = nil)
      if $license.block_changes?
        return build_status_object(false, "License expired")
      end
    end
  end
end

puts "This instance of GitLab Enterprise Edition is licensed to:"
$license.licensee.each do |key, value|
  puts "#{key}: #{value}"
end

if $license.expired?
  puts "The license expired on #{$license.expires_at}"
elsif $license.will_expire?
  puts "The license will expire on #{$license.expires_at}"
else
  puts "The license will never expire."
end
```
создать сертификат
```
ruby license.rb
```
Вот тут нужно попробовать сделать докерфайл
Замените открытый ключ по умолчанию
```
cp -f license_key.pub /opt/gitlab/embedded/service/gitlab-rails/.license_encryption_key.pub
```
Обновление до версии ULTIMATE
Измените соответствующую строку файла

```
--- /opt/gitlab/embedded/service/gitlab-rails/ee/app/models/license.rb
+++ /opt/gitlab/embedded/service/gitlab-rails/ee/app/models/license.rb
@@ -367,7 +367,7 @@
  end

  def plan
-    restricted_attr(:plan).presence || STARTER_PLAN
+    restricted_attr(:plan).presence || ULTIMATE_PLAN
  end

  def edition
```

перенастроить gitlab
```
gitlab-ctl reconfigure
gitlab-ctl restart
```

Войдите в фоновый режим gitlab, центр управления -> лицензия (/admin/license), импортируйте GitLabBV.gitlab-license



