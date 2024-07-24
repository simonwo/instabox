require 'cgi/util'
require 'erb'
require 'json'

rule '.json' => ['.json.xz'] do |t|
  sh "xz -dc #{t.source} > #{t.name}"
end

URL_REGEX = /(?:(?:https?|ftp):\/\/)(?:\S+(?::\S*)?@)?(?:(?!(?:10|127)(?:\.\d{1,3}){3})(?!(?:169\.254|192\.168)(?:\.\d{1,3}){2})(?!172\.(?:1[6-9]|2\d|3[0-1])(?:\.\d{1,3}){2})(?:[1-9]\d?|1\d\d|2[01]\d|22[0-3])(?:\.(?:1?\d{1,2}|2[0-4]\d|25[0-5])){2}(?:\.(?:[1-9]\d?|1\d\d|2[0-4]\d|25[0-4]))|(?:(?:[a-z\u00a1-\uffff0-9]-*)*[a-z\u00a1-\uffff0-9]+)(?:\.(?:[a-z\u00a1-\uffff0-9]-*)*[a-z\u00a1-\uffff0-9]+)*(?:\.(?:[a-z\u00a1-\uffff]{2,}))\.?)(?::\d{2,5})?(?:[\/?#]\S*)?/
USERNAME_REGEX = /\B@([\w.]+\w)/i
TAG_REGEX = /\B#(\w+)/i

def parse text
  escaped = ERB::Util.html_escape text
  escaped.gsub(URL_REGEX, '<a href="\0">\0</a>').
          gsub(USERNAME_REGEX, '<a href="https://instagram.com/\1">\0</a>').
          gsub(TAG_REGEX, '<a href="/?tag=\1">\0</a>')
end

def post source
  data = JSON.parse File.read(source), object_class: OpenStruct
  data.href = source.pathmap('%n.html')

  images = Rake::FileList[source.pathmap('%X*.jpg')].exclude(/_thumb.jpg/)
  data.images = images.pathmap('%f')

  data.video = Rake::FileList[source.pathmap('%X*.mp4')].pathmap('%f')
  data.thumb = source.pathmap('%n_thumb.jpg')

  text = source.pathmap('%X.txt')
  data.text = CGI.unescapeHTML(if File.exist?(text) then File.read(text) else "" end)
  data.title = data.text.scan(/^[\w ]{4,}/).first
  data.tags = data.text.scan(TAG_REGEX).map(&:first).map(&:downcase).uniq
  data.words = data.text.scan(/\w{4,}/).map(&:downcase).uniq - data.tags
  data.text = parse(data.text)
  data
end

DATA_DIR = ENV.fetch("DATA_DIR", Dir.pwd)
POST_TEMPLATE_NAME = "post.html.erb"
POST_TEMPLATE = ERB.new File.read File.join(File.dirname(__FILE__), POST_TEMPLATE_NAME)

rule '.html' => '.json' do |t|
  rake_output_message "erb #{t.source} #{POST_TEMPLATE_NAME} > #{t.name}"
  File.write t.name, POST_TEMPLATE.result_with_hash(post t.source)
end

rule(/_thumb.jpg/ => proc {|n| Rake::FileList[n.gsub(/_thumb.jpg/, '*.jpg')].first}) do |t|
  sh "sips -Z 100 #{t.source} --out #{t.name}"
end

ALL_COMPRESSED_POSTS = Rake::FileList[File.join(DATA_DIR, '*.json.xz')]
ALL_POSTS = ALL_COMPRESSED_POSTS.pathmap("%X")
ALL_HTMLS = ALL_POSTS.pathmap("%X.html")
ALL_THUMBS = ALL_POSTS.pathmap('%X_thumb.jpg')

INDEX_TEMPLATE_NAME = "index.html.erb"
INDEX_TEMPLATE_PATH = File.join(File.dirname(__FILE__), INDEX_TEMPLATE_NAME)
INDEX_TEMPLATE = ERB.new File.read INDEX_TEMPLATE_PATH
INDEX_FILE = File.join(DATA_DIR, 'index.html')

file INDEX_FILE => [*ALL_HTMLS, *ALL_THUMBS, INDEX_TEMPLATE_PATH] do |t|
  all_data = ALL_POSTS.map &method(:post)
  rake_output_message "erb #{INDEX_TEMPLATE_NAME} > #{t.name}"
  File.write t.name, INDEX_TEMPLATE.result_with_hash(posts: all_data)
end

multitask html: [*ALL_HTMLS, INDEX_FILE]

task :clean do
  rm Rake::FileList[INDEX_FILE].existing
  rm ALL_HTMLS.existing
end

task :mrproper do
  rm ALL_POSTS.existing
  rm ALL_THUMBS.existing
end
