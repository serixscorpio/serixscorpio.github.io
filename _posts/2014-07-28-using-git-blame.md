---
layout: post
title: Using git-blame to Understand the Intention of Code
---
While reading through Figaro's [env_spec.rb](https://github.com/laserlemon/figaro/blob/master/spec/figaro/env_spec.rb), I came across this test:

```ruby
describe "#method_missing" do
  ...
  context "plain methods" do
    ...
    it "respects a stubbed plain method" do
      env.stub(bar: "baz")
      expect(env.bar).to eq("baz")
    end
    ...
  end
end
```
At first sight, this test doesn't seem to be doing much.  I could not figure out what the `"respects a stubbed plain method"` means, but it most likely is testing some specific behavior in Figaro.  Fortunately, Joost Baaij, my mentor on Ruby and Rails, pointed out using [`git blame`](http://git-scm.com/docs/git-blame) to understand the intention of this piece of code:

```
> git blame spec/figaro/env_spec.rb -L 32
da9ec5cd (Steve Richert 2014-04-17 15:24:52 -0400  32)       it "respects a stubbed plain method" do
da9ec5cd (Steve Richert 2014-04-17 15:24:52 -0400  33)         env.stub(bar: "baz")
da9ec5cd (Steve Richert 2014-04-17 15:24:52 -0400  34)         expect(env.bar).to eq("baz")
da9ec5cd (Steve Richert 2014-04-17 15:24:52 -0400  35)       end
4655694d (Steve Richert 2013-05-01 12:41:26 -0700  36)     end
4655694d (Steve Richert 2013-05-01 12:41:26 -0700  37) 
...
```
I used git blame with the `-L 32` option to only show change history of line 32 in [env_spec.rb](https://github.com/laserlemon/figaro/blob/master/spec/figaro/env_spec.rb).  Alternatively, one can also use github to get the equivalent [result](https://github.com/laserlemon/figaro/blame/master/spec/figaro/env_spec.rb#L32).  In both cases, we see [da9ec5cd](https://github.com/laserlemon/figaro/commit/da9ec5cdb8c1cc2546ce230aa075bcefc7c79640) as the lastest commit that modified line 32.
```
> git log -L 32,32:spec/figaro/env_spec.rb da9ec5cd
commit da9ec5cdb8c1cc2546ce230aa075bcefc7c79640
Author: Steve Richert <steve.richert@gmail.com>
Date:   Thu Apr 17 15:24:52 2014 -0400

    Make Figaro::ENV bang/boolean methods make use of the plain methods
    
    This allows a developer to stub the plain method and the corresponding
    bang/boolean methods will work as expected. Otherwise, the developer
    would have to stub all three methods separately.

diff --git a/spec/figaro/env_spec.rb b/spec/figaro/env_spec.rb
--- a/spec/figaro/env_spec.rb
+++ b/spec/figaro/env_spec.rb
@@ -31,0 +32,1 @@
+      it "respects a stubbed plain method" do
```
Here I use `git log` with the `-L` option and specified the commit found from `git blame`.  The `-L` option allows me to specify the file and line number I am interested in.

Let's take a look at the commit message.  Ah! Mystery solved.  This test is to ensure Figaro knows how to handle messages `bar`, `bar!`, and `bar?` when a single`stub(bar: "baz")` is defined.  This behavior alleviates the user from having to define three separate methods when stubbing (one plain method, one bang method, and one boolean method).

Finally, this example highlights the importance of a good commit message.  When done right, as in this case, it communicates the intention of the code to other very effectively.