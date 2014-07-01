require 'rubygems'
require 'bundler/setup'

require 'rake/clean'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user root --rpm-group root "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user root --deb-group root "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

language_name = 'php'
license = 'http://php.net/license/'

{
  '5.5.14' => {:md5sum => 'b34262d4ccbb6bef8d2cf83840625201'},
  '5.4.30' => {:md5sum => '461afd4b84778c5845b71e837776139f'},
  '5.3.28' => {:md5sum => 'eec3fb5ccb6d8c238f973d306bebb00e'},
}.each do |version, opts|
  namespace version do
    release = Time.now.utc.strftime('%Y%m%d%H%M%S')
    tarball = "#{language_name}-#{version}.tar.gz"

    configure_flags = %w(
      --disable-debug
      --disable-rpath
      --disable-static
      --enable-bcmath
      --enable-calendar
      --enable-ctype
      --enable-dba
      --enable-dom
      --enable-exif
      --enable-fileinfo
      --enable-filter
      --enable-fpm
      --enable-ftp
      --enable-gd-native-ttf
      --enable-hash
      --enable-inline-optimization
      --enable-json
      --enable-libxml
      --enable-mbregex
      --enable-mbstring
      --enable-pcntl
      --enable-pdo
      --enable-phar
      --enable-posix
      --enable-re2c-cgoto
      --enable-session
      --enable-shared
      --enable-shmop
      --enable-simplexml
      --enable-soap
      --enable-sockets
      --enable-spl
      --enable-sqlite-utf8
      --enable-sysvmsg
      --enable-sysvsem
      --enable-sysvshm
      --enable-tokenizer
      --enable-wddx
      --enable-intl
      --enable-xml
      --enable-xmlreader
      --enable-xmlwriter
      --enable-zip
      --with-bz2
      --with-cdb
      --with-curl
      --with-db4
      --with-enchant
      --with-flatfile
      --with-freetype-dir
      --with-gd
      --with-gettext
      --with-gmp
      --with-gnu-ld
      --with-iconv
      --with-imap-ssl
      --with-imap
      --with-inifile
      --with-interbase
      --with-jpeg-dir
      --with-kerberos
      --with-ldap-sasl
      --with-ldap
      --with-libedit
      --with-mcrypt
      --with-mhash
      --with-mysql=mysqlnd
      --with-mysqli=mysqlnd
      --with-openssl
      --with-pcre-regex
      --with-pdo-firebird
      --with-pdo-mysql=mysqlnd
      --with-pdo_sqlite
      --with-pgsql
      --with-pic
      --with-png-dir
      --with-pspell
      --with-sqlite3
      --with-tidy
      --with-xmlrpc
      --with-xsl
      --with-zlib-dir
      --with-zlib
      --without-gdbm
    )

    configure_flags += %w(
      --with-libdir=/usr/lib64
      --with-libdir=lib64
    )

    description_string = %Q{PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated webpages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.}

    jailed_root = File.expand_path('../jailed-root', __FILE__)
    prefix = File.join(%Q{/opt/local/#{language_name}}, version.split('.').first(2).join('.'))

    CLEAN.include("downloads")
    CLEAN.include("jailed-root")
    CLEAN.include("log")
    CLEAN.include("pkg")
    CLEAN.include("src")

    task :init do
      mkdir_p "log"
      mkdir_p "pkg"
      mkdir_p "src"
      mkdir_p "downloads"
      mkdir_p "jailed-root"
    end

    task :download do
      cd 'downloads' do
        sh("curl --fail --location http://www.php.net/get/#{tarball}/from/this/mirror > #{tarball} 2>/dev/null")
        File.open("#{tarball}.md5sum", 'w') {|f| f.puts("#{opts[:md5sum]}  #{tarball}")}
        sh("md5sum --status --check #{tarball}.md5sum")
      end
    end

    task :configure do
      cd "src" do
        sh "tar -zxf ../downloads/#{tarball}"
        cd "#{language_name}-#{version}" do
          sh "./configure --prefix=#{prefix} #{configure_flags.join(' ')} > #{File.dirname(__FILE__)}/log/configure.#{version}.log 2>&1"
        end
      end
    end

    task :make do
      num_processors = %x[nproc].chomp.to_i
      num_jobs       = num_processors + 1

      cd "src/#{language_name}-#{version}" do
        sh("make -j#{num_jobs} > #{File.dirname(__FILE__)}/log/make.#{version}.log 2>&1")
      end
    end

    task :make_install do
      rm_rf  jailed_root
      mkdir_p jailed_root
      cd "src/#{language_name}-#{version}" do
        sh("make install DESTDIR=#{jailed_root} > #{File.dirname(__FILE__)}/log/make-install.#{version}.log 2>&1")
      end
    end

    task :fpm do
      cd 'pkg' do
        sh(%Q{
             bundle exec fpm -s dir -t #{distro} --name #{language_name}-#{version} -a x86_64 --version "#{version}" -C #{jailed_root} --directories #{prefix} --verbose #{fpm_opts} --maintainer suppport@snap-ci.com --vendor suppport@snap-ci.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license '#{license}' .
        })
      end
    end

    desc "build and package #{language_name}-#{version}"
    task :all => [:clean, :init, :download, :configure, :make, :make_install, :fpm]
  end

  task :default => "#{version}:all"
end

desc "build all #{language_name}"
task :default
