desc "Clean the site"
task :clean do
  rm_rf "_site"
end

desc "Generate the site using Jekyll"
task :generate do
  ruby "-S bundle exec jekyll"
end
task :gen => :generate

desc "Run a file server that serves and regenerates the files"
task :server  do
  ruby "-S bundle exec rackup"
end

desc "Deploy the files to jruby.org"
task :deploy => :generate do
  sh "tar -C _site/ -zcf - . | ssh deploy@jruby.org 'cd /data/jruby.org && tar zxf -'"
end

task :default do
  puts "JRuby.org documentation site. Available tasks:"
  Rake.application.options.show_tasks = true
  Rake.application.options.full_description = false
  Rake.application.options.show_task_pattern = //
  Rake.application.display_tasks_and_comments
end

def manifest_xml(stream = File.new('s3manifest.xml'))
  require 'rexml/document'
  @doc ||= REXML::Document.new(stream)
end

def manifest_entries
  manifest_xml.root.elements.to_a('/ListBucketResult/Contents/Key')
end

file 's3manifest.xml' do |t|
  require 'open-uri'
  open("http://jruby.org.s3.amazonaws.com/") do |xml|
    File.open(t.name, "wb") {|f| manifest_xml(xml).write(f, 2) }
  end
end

desc "Create browsable index.html files for S3"
task :indexes => 's3manifest.xml' do
  entries = manifest_entries.map do |el|
    el.text.strip.sub(/_\$folder\$$/, '/')
  end
  dirs = {"." => []}
  entries.sort.each do |f|
    dirs[File.dirname(f)] ||= []
    dirs[File.dirname(f)] << f
  end
  top = "www/files"
  mkdir_p top, :verbose => false
  dirs.each do |dir,entries|
    mkdir_p File.expand_path(File.join(top, dir)), :verbose => false
    File.open(File.expand_path(File.join(top, dir, "index.html")), "wb") do |html|
      html.puts <<HDR
---
layout: main
title: Files/#{dir == '.' ? '' : dir}
---
<h1>Files/#{dir == '.' ? '' : dir}</h1>
<p class="trackDownloads">
HDR
      parent = File.dirname(dir)
      parent = parent == '.' ? '' : "#{parent}/"
      html.puts "  <a href='/files/#{parent}index.html'>..</a><br/>" unless dir == '.'
      entries.sort.each do |entry|
        if entry =~ /\/$/
          html.puts "  <a href='/files/#{entry}index.html'>#{File.basename(entry)}</a><br/>"
        else
          html.puts "  <a href='http://jruby.org.s3.amazonaws.com/#{entry}'>#{File.basename(entry)}</a><br/>"
        end
      end
      html.puts "</p>"
    end
  end
end

def jruby_org_bucket
  require 'aws/s3'
  ey_cloud = open(File.expand_path('~/.ey-cloud.yml')) { |f| YAML::load(f) }
  AWS::S3::Base.establish_connection!(
    :access_key_id     => ey_cloud[:aws_secret_id],
    :secret_access_key => ey_cloud[:aws_secret_key]
  )
  AWS::S3::Bucket.find('jruby.org')
end

def add_public_read_perm(obj)
  return if obj.acl.grants.detect {|g| g.grantee.group == "AllUsers" && g.permission == "READ" }
  puts "Updating #{obj.key} to be publicly readable"
  obj.acl.grants << AWS::S3::ACL::Grant.grant(:public_read)
  obj.acl(obj.acl)              # save the permissions
end

task :update_hash_files do
  jruby_org_bucket.objects.each do |obj|
    next unless obj.key =~ /\.(md5|sha1)$/
    unless obj.content_type == "text/plain"
      puts "Updating #{obj.key} to Content-Type: text/plain"
      obj.content_type = "text/plain"
      obj.store
      add_public_read_perm(obj)
    end
  end
end

task :update_read_perms do
  jruby_org_bucket.objects.each do |obj|
    add_public_read_perm(obj)
  end
end
