---
layout: post
title: 基于GNOME的Evolution接收邮件优化
description: "2012年文章，收录过来"
modified: 2012-05-29
categories: [Linux]
type: dev
---

系统S3期间网络连接将断开，收取邮件的动作发生在唤醒电脑并重新连接网络后，此优化目的为系统S3期间唤醒网络，完成邮件收取，唤醒电脑时邮件已在列表中。

<!-- more -->

- ##### 方案选择
  dbus需要loop等待，evolution会hold，不能用此方式。该需求用signal方式对发射收集安装都能很好的控制，固采用signal。

- ##### 实现
  实现该功能之前，先看看系统代码 ，了解下GNOME是如何完成邮件收取的（因该feature与发送邮件无关，无需关注）。代码路径：evolution/src/mail/mail-send-recv.c ，发现邮件的发送通过mail_receive接口完成。
  {% highlight c %}
    GtkWidget *
    mail_receive (GtkWindow *parent,
              EMailSession *session)
    {
      return send_receive (parent, session, FALSE);
    }
  {% endhighlight %}
  再在e-shell.c中new一个signal，并emit，在mail/mail-send-rece.c中connect此信号调用接收邮件接口，
  在脚本中通过参数调用时调用main函数中一个handle接口,在e-shell.c中判断shell是否remote，在通过g_action_group_activate_action激活自动接收邮件的action：
  {% highlight c %}
  void e_shell_handle_mail_rec (EShell *shell)
  {
  		if (g_application_get_is_remote (G_APPLICATION 				(shell))){
  			g_action_group_activate_action(G_ACTION_GROUP 				(shell), "automatic", NULL);
  			}
  }
  {% endhighlight %}
  用类似--help的带参数启动方式，在启动脚本中加入此命令。