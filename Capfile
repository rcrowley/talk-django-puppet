set :remote, "/home/rcrowley/work/django-2010-11-17"

set :user, "rcrowley"
role :www, "rcrowley.org"

task :static do
  system "showoff static"
end

task :deploy do
  static
  run "rm -rf #{remote}"
  upload "static", remote
  Dir["*.css", "*.js"].each do |filename|
    upload filename, "#{remote}/#{filename}"
  end
end
