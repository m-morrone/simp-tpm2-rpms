require 'rake/tasklib'
require 'fileutils'
require 'rake/clean'
require 'yaml'



module SIMP; end
module SIMP::RPM; end
class SIMP::RPM::SpecBuilder < Rake::TaskLib

  if Rake.verbose
    include FileUtils::Verbose
  else
    include FileUtils
  end

  # This method exists because `vagrant up` dereferences symlinks
  def yaml_config_path

    file_name = 'things_to_build.yaml'
    _dir = File.dirname File.expand_path *Rake.application.find_rakefile_location
    _yaml_file = nil
    while _yaml_file.nil? && _dir !~ /^\/$/
      _file = File.join(_dir,file_name)
      if File.file? _file
        _yaml_file = _file
        break
      else
        _dir = File.dirname _dir
      end
    end
    fail "ERROR: couldn't find #{file_name}" unless _yaml_file
    _yaml_file
  end

  def initialize
    @things_to_download = YAML.load_file( yaml_config_path )
    @dirs = {}
    @dirs[:dist]               = File.expand_path('dist')
    @dirs[:rpmbuild]           = File.expand_path('rpmbuild',@dirs[:dist])
    @dirs[:tmp]                = File.expand_path('tmp',@dirs[:dist])
    @dirs[:logs]               = File.expand_path('tmp',@dirs[:tmp])
    @dirs[:rpmbuild_sources]   = File.expand_path('SOURCES',@dirs[:rpmbuild])
    @dirs[:rpmbuild_build]     = File.expand_path('BUILD',@dirs[:rpmbuild])
    @dirs[:rpmbuild_buildroot] = File.expand_path('BUILDROOT',@dirs[:rpmbuild])
    @dirs[:extra_sources_dir]  = File.expand_path('extra_sources',@dirs[:dist])
  end

  def dl_untar(url,dst)
    mkdir_p dst
    Dir.chdir dst
    sh "curl -sSfL '#{url}' | tar zxvf -"
  end

  def git_clone(url,ref,dst)
    sh "git clone '#{url}' -b '#{ref}' '#{dst}'"
  end


  # Downloads via git clone or URL for targz
  def download( url, dir, type, tag=nil )
    url = url.gsub('%{TAG}',tag) if tag
    Dir.chdir File.dirname(dir)
    if File.directory? dir
      warn "WARNING: path '#{dir}' already exists; aborting download"
      return dir
    end
    case type
    when :targz
      dl_untar url, dir
    when :gitrepo
      git_clone url, tag, dir
    else
      fail "ERROR: :type is not :targz or :gitrepo (#{dl_info.inspect})"
    end
    dir
  end


  def tar_gz key, thing, dst
    _parent, _dir = File.dirname(dst), File.basename(dst)
    cwd = Dir.pwd
    Dir.chdir(_parent)
    sh "tar zcvf '#{_dir}.tar.gz' '#{_dir}'"
    Dir.chdir cwd
  end


  def spec_info spec_file
    info = {}
    cmd="rpm -q --define 'debug_package %{nil}' --queryformat '%{NAME} %{VERSION} %{RELEASE} %{ARCH}\n' --specfile '#{spec_file}'"
    cmd="rpm -q --define 'debug_package %{nil}' --queryformat '%{NAME} %{VERSION} %{RELEASE} %{ARCH}\n' --specfile '#{spec_file}'"
    info[:basename], info[:version], info[:release], info[:arch] = %x{#{cmd}}.strip.split
    info[:full_name]="#{info[:basename]}-#{info[:version]}-#{info[:release]}"
    info[:ver_name]="#{info[:basename]}-#{info[:version]}"
    info
  end

  CLEAN << 'dist'


  def mk_dirs
    @dirs.each do |k,dir|
      mkdir_p dir
    end
  end

  def _download(spec,cwd)
    spec_path = File.expand_path(spec)
    spec_path = File.expand_path( spec, cwd)
    info      = spec_info(spec_path)
    dl_dir    = File.expand_path("dist/#{info[:ver_name]}")
    dl_info   = @things_to_download[info[:basename]]

    # download the source0
    download(dl_info[:url], dl_dir, dl_info[:type], dl_info[:tag])

    # download extras (source1, etc)
    Dir.chdir dl_dir
    cmds = dl_info.fetch(:extras,{}).fetch(:post_dl,[])
    cmds.each{ |cmd| sh cmd.gsub('%{SOURCES_DIR}',@dirs[:extra_sources_dir]) }
  end

  # All in one go because there's no time to be fancy this sprint
  def _rpm(spec,cwd)
    spec_path = File.expand_path( spec, cwd)
    info      = spec_info(spec_path)
    dl_dir    = File.expand_path("#{info[:ver_name]}",@dirs[:dist])
    dl_info   = @things_to_download[info[:basename]]

    Dir.chdir File.dirname(dl_dir)
    tar_file = File.join(@dirs[:rpmbuild_sources], "#{info[:ver_name]}.tar.gz")
    puts "===================================== TAR ============================\n" * 7
    # Build process got cranky without .git to reference
    ###tar_cmd='tar --owner 0 --group 0 --exclude-vcs ' \
    tar_cmd='tar --owner 0 --group 0 ' \
      "-cpzf #{tar_file} #{File.basename dl_dir}"
    sh tar_cmd
    puts "------------------- cp -r #{File.join(@dirs[:extra_sources_dir],'.')} #{@dirs[:rpmbuild_sources]}"
    FileUtils.cp_r(File.join(@dirs[:extra_sources_dir],'.'), @dirs[:rpmbuild_sources])

    Dir.chdir cwd
    puts "===================================== SRPM ============================\n" * 7
    srpm_cmd="RPM_BUILD_ROOT=#{@dirs[:rpmbuild_buildroot]} "\
      "rpmbuild -D 'debug_package %{nil}' " \
      "-D '_topdir #{@dirs[:rpmbuild]}' " \
      "-D '_rpmdir #{@dirs[:dist]}' -D '_srcrpmdir #{@dirs[:dist]}' " \
      "-D '_build_name_fmt %%{NAME}-%%{VERSION}-%%{RELEASE}.%%{ARCH}.rpm' " \
      " -v -bs '#{spec_path}' " \
      "|& tee #{@dirs[:logs]}/build.srpm.log"
    File.open(File.join(@dirs[:tmp],'srpm.sh'),'w'){|f| f.puts srpm_cmd }
    sh srpm_cmd

    Dir.chdir cwd
    puts "===================================== RPM ============================\n" * 7
    rpm_cmd="RPM_BUILD_ROOT=#{@dirs[:rpmbuild_buildroot]} "\
      "rpmbuild --define 'debug_package %{nil}' " \
      "-D '_topdir #{@dirs[:rpmbuild]}' " \
      "-D '_rpmdir #{@dirs[:dist]}' -D '_srcrpmdir #{@dirs[:dist]}' " \
      "-D '_build_name_fmt %%{NAME}-%%{VERSION}-%%{RELEASE}.%%{ARCH}.rpm' " \
      " -v -ba #{spec_path}  " \
      "|& tee #{@dirs[:logs]}/build.rpm.log"

    File.open(File.join(@dirs[:tmp],'rpm.sh'),'w'){|f| f.puts rpm_cmd }
    sh rpm_cmd
  end

  def define_tasks
    cwd = Dir.pwd

    Dir[ '*.spec' ].each do |spec|
      spec_path = File.expand_path(spec)
      info = spec_info(spec)
      dl_dir = File.expand_path("dist/#{info[:ver_name]}")

      namespace :src do
        task :mkdirs do
          mk_dirs
        end

        desc 'downloads source'
        task :download => ['src:mkdirs'] do
          _download(spec,cwd)
        end
      end

      namespace :pkg do
        desc 'builds RPM'
        task :rpm => ['src:mkdirs','src:download'] do
          _rpm(spec,cwd)
        end
      end
    end
  end

end

builder = SIMP::RPM::SpecBuilder.new

builder.define_tasks

