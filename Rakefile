require 'rake-jekyll'

# This task builds the Jekyll site and deploys it to a remote Git repository.
# It's preconfigured to be used with GitHub and Travis CI.
# See http://github.com/jirutka/rake-jekyll for more options.
Rake::Jekyll::GitDeployTask.new(:deploy) do |t|
    t.committer = 'Haytham Mohamed <haybu@hotmail.com>'
    #t.commit_message = -> {
      #"Built from #{`git rev-parse --short HEAD`.strip}"
    #}
    #t.author = -> {
      #`git log -n 1 --format='%aN <%aE>'`.strip
    #}
    t.build_script = ->(dest_dir) {
      puts "\nRunning Jekyll..."
      sh "bundle exec jekyll build --verbose --destination $PWD/_site && git add --all && git commit --author='Haytham Mohamed <haybu@hotmail.com>' -m 'update'"
    }
    #t.deploy_branch = 'gh-pages'
    
end
