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

class RpmSpec < Struct.new(:version, :release, :prefix, :conf_dir, :conf_dir_includes)
  def get_binding
    binding
  end
end


{
  '5.5.15' => {:md5sum => '63b56e64e7c25b1c6dcdf778333dfa24'},
  '5.4.31' => {:md5sum => '07985cff81820666fbf0b0c46f5d35df'},
  '5.3.28' => {:md5sum => 'eec3fb5ccb6d8c238f973d306bebb00e'},
  '5.6.0RC2' => {:md5sum => '99769c4c3477168e0b96d1f228ad23fb'}
}.each do |version, opts|
  namespace version do
    release = Time.now.utc.strftime('%Y%m%d%H%M%S')
    tarball = "#{language_name}-#{version}.tar.gz"

    description_string = %Q{PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated webpages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.}

    jailed_root       = File.expand_path('../jailed-root', __FILE__)
    prefix            = File.join("/opt/local/#{language_name}", version)
    conf_dir          = File.join(prefix, 'etc')
    conf_dir_includes = File.join(prefix, 'etc', 'conf.d')

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

    configure_flags += %W(
      --with-libdir=lib64
      --with-config-file-path=#{conf_dir}
      --with-config-file-scan-dir=#{conf_dir_includes}
    )

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
        if tarball == 'php-5.6.0RC2.tar.gz'
          sh("curl --fail --location http://downloads.php.net/tyrael/#{tarball} > #{tarball} 2>/dev/null")
        else
          sh("curl --fail --location http://www.php.net/get/#{tarball}/from/this/mirror > #{tarball} 2>/dev/null")
        end
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
        # sh("make test -j#{num_jobs} > #{File.dirname(__FILE__)}/log/make.test.#{version}.log 2>&1")
      end
    end

    task :make_install do
      rm_rf  jailed_root
      mkdir_p jailed_root
      cd "src/#{language_name}-#{version}" do
        sh("make install INSTALL_ROOT=#{jailed_root} > #{File.dirname(__FILE__)}/log/make-install.#{version}.log 2>&1")
        jailed_conf_dir          = File.join(jailed_root, conf_dir)
        jailed_conf_dir_includes =  File.join(jailed_root, conf_dir_includes)

        mkdir_p jailed_conf_dir
        mkdir_p jailed_conf_dir_includes

        cp "php.ini-development", "#{jailed_conf_dir}/php.ini"
        cp File.join(jailed_conf_dir, 'php-fpm.conf.default'), File.join(jailed_conf_dir, 'php-fpm.conf')

        rm_rf (Dir["#{jailed_root}/.*"] - Dir["#{jailed_root}/{.,..}"])
      end
    end

    task :rpm do
      require 'erb'

      ERB.new(File.read(File.expand_path('../php.spec.erb', __FILE__)), nil , '-').tap do |template|
        File.open("/tmp/php.spec", 'w') do |f|
          attrs = RpmSpec.new(version, release, prefix, conf_dir, conf_dir_includes)
          f.puts(template.result(attrs.get_binding))
        end
        at_exit {rm_rf "/tmp/php.spec"}
      end

      cd jailed_root do
        output_dir = File.expand_path('../target-rpms', __FILE__)
        at_exit {rm_rf output_dir}
        puts "*** Building RPM..."
        rpmbuild_cmd = []
        rpmbuild_cmd << "rpmbuild /tmp/php.spec"
        rpmbuild_cmd << "--verbose"
        rpmbuild_cmd << "--buildroot #{jailed_root}"
        rpmbuild_cmd << "--define '_tmppath #{jailed_root}/../rpm_tmppath'"
        rpmbuild_cmd << "--define '_topdir #{output_dir}'"
        rpmbuild_cmd << "--define '_rpmdir #{output_dir}'"
        rpmbuild_cmd << "--define 'noclean 1'"
        rpmbuild_cmd << "-bb"

        sh rpmbuild_cmd.join(" ")
      end
      sh("mv target-rpms/x86_64/*.rpm pkg/")
    end

    desc "build and package #{language_name}-#{version}"
    task :all => [:clean, :init, :download, :configure, :make, :make_install, :rpm]
  end

  task :default => "#{version}:all"
end

desc "build all #{language_name}"
task :default
